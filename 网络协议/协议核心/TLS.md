---
title: TLS 加密协议完整技术手册（TLS 1.2 / 1.3）
tags:
  - Network
  - Protocol
  - TLS
  - Cryptography
  - HTTPS
status: Completed
created: 2026-09-21
---

# 🔐 TLS 加密协议完整技术手册（TLS 1.2 / 1.3）

> [!INFO] 笔记概览
> - **协议定位**：TLS（Transport Layer Security）位于传输层与应用层之间，给任意基于 TCP 的应用协议（HTTP/[[http]]、[[FTP]]S、[[MQTT]]S、[[SSH]] 之外另起炉灶）加装**机密性、完整性、服务端（可选双向）身份认证**，解决明文协议被窃听/篡改/冒充三大问题。
> - **内容范围**：密码学构件 · 记录层帧格式 · 握手协议（1.3 与 1.2 对比）· 密钥推导 · 证书链验证 · 会话恢复与 0-RTT · Alert · 部署排错
> - **推荐路径**：§1 构件 → §2 记录层 → §3 1.3 握手 → §4 1.2 差异 → §5 证书 → §6 密钥体系 → §7 恢复/性能 → §8 排错清单

---

## 🧱 1. 协议靠什么撑起安全：密码学构件

TLS 不是自造算法，而是四件标准密码学工具的**组装协议**：

| 构件                  | 作用               | TLS 1.3 允许的代表算法                                                |
| :------------------ | :--------------- | :------------------------------------------------------------- |
| **认证公钥（签名）**        | 证明"我持有私钥"，防中间人   | ECDSA-P256、RSA-PSS、Ed25519                                     |
| **密钥协商（(K)EM/KEX）** | 双方在不安全信道上算出共享秘密  | (EC)DHE（**1.3 唯一**：X25519、secp256r1）、Hybrid（ML-KEM，后量子，1.3 扩展） |
| **AEAD 对称加密**       | 加密+完整性一体，握手后全程使用 | AES-128/256-GCM、ChaCha20-Poly1305                              |
| **HKDF / 哈希**       | 从共享秘密扩展出各方向各用途密钥 | HKDF-SHA256/SHA384；SHA-256/384（**1.3 握手内哈希全部 AEAD 化之前均为标准套件**） |

> [!WARNING] 1.3 已删除的危险部件
> RSA 密钥传输（无前向保密）、CBC 模式、RC4、SHA-1、MD5、3DES、压缩（CRIME 攻击）、export 级密码套件在 TLS 1.3 中全部移除。看到 `TLS_RSA_WITH_AES_128_CBC_SHA` 即知对端只支持老协议。

**前向保密（PFS）**： ephemeral DHE/ECDHE 的临时私钥用完即毁——长期证书私钥日后泄露也无法解密历史流量。这是 1.3 强制 DHE 式协商的根本原因。

---

## 📦 2. 记录层（Record Layer）：唯一的封装载体

所有握手/告警/应用数据都被装进同一种"信封"：

### 2.1 记录层帧格式

```text
┌──────────────┬─────────────┬──────────────┬─────────────────────┐
│ContentType   │ Version     │ Length       │ Fragment            │
│ 1 B          │ 2 B         │ 2 B          │ ≤ 2^14 B (16KB)     │
│ 20/21/22/23/ │ 0x0303=1.2  │ 载荷长度      │ 明文 或 AEAD 密文    │
│ 254(CCS)     │ 0x0304=1.3  │              │ +1.3记录尾部Type    │
└──────────────┴─────────────┴──────────────┴─────────────────────┘
```

| ContentType          |  值  | 内容                          |
| :------------------- | :-: | :-------------------------- |
| `change_cipher_spec` | 20  | 兼容占位（1.3 仅作中间件兼容桩，单字节 0x01） |
| `alert`              | 21  | 关闭通知/错误告警                   |
| `handshake`          | 22  | 握手协议消息（可分片装进多条记录）           |
| `application_data`   | 23  | 加密后的上层协议字节流                 |

### 2.2 握手消息通用头

```text
┌────────────────┬──────────────────┬──────────────┐
│ Handshake Type │ Length (3 B)     │  Body        │
│ 1 B            │                  │              │
└────────────────┴──────────────────┴──────────────┘
```

|  Type  | 消息                               | 方向（1.3）   | 说明                                            |
| :----: | :------------------------------- | :-------- | :-------------------------------------------- |
|   1    | ClientHello                      | C→S       | 支持的版本/套件/密钥share/SNI/ALPN                     |
|   2    | ServerHello                      | S→C       | 选定套件+服务端密钥share，之后立即转向加密                      |
|   8    | encrypted_extensions             | S→C       | **1.3 新增**：扩展加密后发送                            |
|   11   | certificate                      | S→C / C→S | 证书链（1.3 加密）                                   |
|   15   | CertificateVerify                | S→C / C→S | **对握手 transcript 哈希的私钥签名**（1.3 加密）            |
| 16 / 4 | finished                         | 双向        | **基于握手历史+派生密钥的 HMAC（Finished MAC），完成认证与密钥确认** |
|   1    | session_ticket(NewSessionTicket) | S→C       | 1.3 握手机会后下发恢复凭据                               |
|   20   | EndOfEarlyData                   | C→S       | 0-RTT 结束标记                                    |

> [!NOTE] 1.3 的"加密早"设计
> ServerHello 之后服务端即切到 AEAD：Certificate、CertificateVerify、Finished 都是密文——客户端 IP/SNI 之外的细节（证书内容、扩展）不再裸奔。1.2 里 Certificate 是明文。

### 2.3 AEAD 记录（1.3）

`sequence number（不传输，隐式参与 nonce）+ 0x0000… 填充` 后：

```text
┌───────────────┬──────────────┬─────────┬────────┐
│ AAD=记录头     │ Ciphertext   │ Type(1B)│ Tag(16B)│
│ Version+Len   │ (Fragment)   │ 内层类型 │ AEAD   │
└───────────────┴──────────────┴─────────┴────────┘
nonce = 0x00000000 ‖ IV(12B, 握手/数据各一套) ⊕ seq
```

Type 被藏进密文——外部抓包只能看到 `application_data(23)` 的密文 blob，这正是 1.3 元数据保护。

---

## 🤝 3. TLS 1.3 握手：1-RTT 完整时序

```text
Client                                        Server
  │ ClientHello                                │
  │  supported_versions=0x0304                 │
  │  key_share: x25519: pubKey_C               │
  │  cipher_suites: TLS_AES_128_GCM_SHA256 …   │
  │  server_name: www.example.com (SNI)        │
  │  supported_groups / sig_algs / alpn(h2,1.1)│
  │ ──────────────────────────────────────────► │
  │                          (ECDHE: K = f(pubKey_C, privKey_S))
  │                 ServerHello                │
  │                 key_share: pubKey_S        │
  │ ◄────────────────────────────────────────── │
  │ {EncryptedExtensions}   ← 以下全部加密      │
  │ {Certificate: 链}                           │
  │ {CertificateVerify: sig_priv_cert(H(handshake))]
  │ {Finished: HMAC(finished_key, transcript)}  │
  │ ◄────────────────────────────────────────── │
  │ 校验证书链、校验 CV 与 Finished             │
  │ {Finished}                                  │
  │ ──────────────────────────────────────────► │
  │                 [NewSessionTicket]          │
  │ ◄────────────────────────────────────────── │
  │ ◄═════════应用数据双向═════════►             │
```

### 3.1 三条主线在握手中交织完成

| 目标 | 靠哪条消息 | 机制 |
| :--- | :--- | :--- |
| 协商出会话密钥 | ClientHello/ServerHello 的 key_share | ECDHE → HKDF-Extract/Expand 派生 handshake/app keys |
| 证明服务器身份 | Certificate + CertificateVerify + Finished | 证书链信任锚 + 对**完整握手 transcript** 的签名（防 downgrade/篡改参数） |
| 防握手被篡改 | Finished | `HMAC(finished_key, Transcript-Hash(Hello…最后一条))`，任何中间修改都会使对端校验失败 |

> [!CAUTION] SNI 加密（ECH）
> 明文 ClientHello 中的 `server_name` 仍可被旁观——1.3 后续扩展 **Encrypted Client Hello（ECH）** 把 inner ClientHello（含 SNI）用 HPKE 加密，部署依赖 DNS HTTPS 记录，目前普及度有限。

---

## 🔁 4. TLS 1.2 对照：2-RTT 与已废弃部件

```text
Client                          Server
 ClientHello（legacy_session_id、cipher_suites 含 ECDHE_RSA_* 等）
 ─────────────────────────────► │
                                │ ServerHello
                                │ Certificate（明文！）
                                │ ServerKeyExchange（仅 ECDHE 套件需要）
                                │ ServerHelloDone
 ◄───────────────────────────── │
 ClientKeyExchange（公钥加密 premaster 或 ECDHE 参数）
 ChangeCipherSpec + Finished
 ─────────────────────────────► │
                                │ ChangeCipherSpec + Finished
 ◄───────────────────────────── │
      [会话密钥 = PRF(premaster, "master secret", randoms) → key_block]
```

| 维度 | TLS 1.2 | TLS 1.3 |
| :--- | :--- | :--- |
| RTT | 2-RTT | **1-RTT**（恢复 0-RTT） |
| 密码套件定义 | 密钥交换+签名+加密+MAC 四合一，组合爆炸（~300 种） | **仅 5 种 AEAD 套件**，KEX/签名独立协商 |
| 握手状态 | 消息顺序脆弱、需扩展修补 | 状态机收敛、消息可乱序 |
| RSA 密钥传输 | 可用（无前向保密） | **禁止** |
| 协商失败降级 | HelloRetryRequest 无 | 有（1.2 ClientHello 兼容 Cookie） |
| 证书之后流量 | Certificate 明文 | 全部加密 |
| Downgrade 保护 | 弱 | ServerHello 插入特殊降级信号并计入 Finished |

---

## 🏛️ 5. 证书与身份：链式信任

### 5.1 X.509 v3 证书关键字段

| 字段 | 作用 |
| :--- | :--- |
| Subject / Subject Alternative Name | 域名/实体名；**浏览器只认 SAN**（CN 已废弃匹配） |
| Public Key + Algorithm | 被认证的公钥（网站服务器公钥，≠ 握手临时 ECDHE 公钥） |
| Issuer + Signature | 上一级 CA 私钥对本证书 DER 的签名 → 链 |
| Validity（NotBefore/NotAfter） | 过期即 `ERR_CERT_DATE_INVALID` |
| Basic Constraints / Key Usage | 能否签发下级、用途限定 |
| Serial、Version、Signature Algorithm | 标识与算法 |

### 5.2 验证链（客户端五步）

```text
leaf(服务器) → intermediate CA(服务器随 Certificate 发送) → root CA(操作系统内置信任库)
① 逐级验签  ② 检查有效期/用途/路径长度  ③ 域名匹配 SAN  ④ 吊销状态(CRL/OCSP stapling)
```

- **中间证书漏发是最高频部署故障**：浏览器可缓存补链而 `curl`/设备 SDK 直接失败——`openssl s_client` 必须看完整链。
- **OCSP Stapling**：服务器替客户端附带时间戳签名吊销回执，省一次访问 CA。
- **证书透明度（CT）**：公开可审计日志，Google 证书强制≥2 条 SCT。
- **双向 mTLS**：客户端同样发 Certificate/CertificateVerify（1.3 中作为第二 flight 可选），用于零信任内网/IoT 设备身份，见 §9 嵌入式。

---

## 🔑 6. 密钥体系：从共享秘密到多把钥匙

```text
          ECDHE(pubKey_C, privKey_S) ──► Early/Derive-Secret 阶梯（HKDF-Extract+Expand）
 TLS1.3:
   derived0 = HKDF-Extract(salt=0, IKM=0)            ← "初始化"
   …每步 Derive-Secret(Secret, label, messages) 产出：
     client/server hs traffic keys     → 握手加密（方向独立）
     exporter secret / resumption master → 恢复 PSK、keying material 导出
     application traffic keys            → 数据段 AEAD 密钥+IV
 TLS1.2（理解用）:
   premaster → master_secret（PRF, 双方 random 参与）→ key_block 切 6 份：
   client_write_MAC/key + server_write_MAC/key + 双向 IV
```

要点：
- **方向分离**：上下行各自独立密钥/序号，防反射与序列混淆。
- **Finished 不加密也安全**：finished_key 无法从公网流量反推——同时完成"密钥确认"（key confirmation）。
- **导出器（exporter）**：把 TLS 会话密钥借给上层（EAP-TLS、token binding）而不泄露主密钥。

---

## ⚡ 7. 会话恢复与性能

| 机制 | 版本 | RTT | 重放风险 |
| :--- | :--- | :--- | :--- |
| Session ID 缓存 | 1.2 | 省 1 RTT | — |
| Session Ticket（服务端加密保存密钥） | 1.2 | 省 1 RTT | 票据密钥轮换策略影响 PFS |
| **PSK（NewSessionTicket）** | 1.3 | **0-RTT 恢复** | **0-RTT 数据可被重放，仅幂等请求可用** |
| PSK + DHE（默认建议） | 1.3 | 1-RTT，保留前向保密 | 无 |
| TLS False Start / OCT | 历史 | 试验性 | 已淘汰 |

性能账（1.3 相对 1.2）：握手字节数约 −30%、RTT −1、CPU 侧 AEAD 硬件加速（AES-NI）后几乎白送。移动端"连接复用 > 一切优化"——HTTP keep-alive / h2 多路复用让多数请求零 TLS 握手。

---

## 🚨 8. Alert 与常见故障排查

### 8.1 Alert 记录

`level(1B: warning=1/fatal=2) + description(1B)`；TLS 1.3 中几乎一切错误都是 fatal + 立即关连接，且 Alert 本身被 AEAD 加密。

| 值 | 名称 | 典型根因 |
| :--: | :--- | :--- |
| 40 | handshake_failure | 无交集密码套件/版本（老设备连新服务常见） |
| 46 | certificate_unknown | 链验证失败 |
| 48 | unknown_ca | 根证书不受信（自签/内网 CA 未导入） |
| 70 | protocol_version | 仅支持 SSLv3/TLS1.0 的旧客户端 |
| 80 | internal_error | 对端实现 bug |
| 109/112 | missing_extension / unsupported_certificate | 缺必发扩展/证书类型不对 |
| 120 | no_application_protocol | ALPN 协商失败（HTTP/2 端点被回落到 1.1） |
| 111 | unrecognized_name | SNI 未配置（多租户反代漏配） |

### 8.2 排障命令速查

```bash
# 看协议/套件/证书链/握手耗时
openssl s_client -connect host:443 -servername host -showcerts </dev/null

# 强制版本：定位"是不是只支持老协议"
openssl s_client -connect host:443 -tls1_2   # 或 -tls1_3 / -dtls1_2

# 抓包：1.3 数据段密文，但握手协商过程全可读（含 SNI）
tshark -i any -f "tcp port 443" -Y "tls.handshake"

# ALPN/证书链一键体检
testssl.sh --defaults host
```

常见症状 → 根因对照：

| 症状 | 大概率根因 |
| :--- | :--- |
| 浏览器能开、SDK/curl 报错 | 中间证书漏发（见 §5.2） |
| `unable to get local issuer certificate` | 自签根未导入设备信任库/系统 CA 太旧 |
| 证书未过期仍 `CERT_DATE_INVALID` | 设备 RTC 时钟漂移（IoT 高发） |
| 握手偶发 `sslv3 alert handshake failure` | 客户端不支持 1.3，服务禁用了兼容层 |
| 只连得上 IP 连不上域名 | SNI 未配置 / ECH 中间设备干扰 |
| DTLS 丢包后整链卡死 | replay window 与 MTU 配置（DTLS 对照见 §9） |

---

## 🧊 9. 嵌入式视角：mbedTLS / wolfSSL

- MCU 上 TLS 是**RAM 大户**：1.3 握手 transcript 哈希需要缓冲完整握手消息（默认 `MAX_CONTENT_LEN=16KB`），mbedTLS 可把收发缓冲砍到 2–6KB（限制记录长度，RFC 8446 §5.4 允许）。
- 硬件：`MBEDTLS_AES_ROM_TABLES`/`MBEDTLS_ECDH_VARIANT_EVEREST_ENABLED`、有无 PSA Crypto 硬件加速决定 CPU 开销数量级。
- **DTLS 1.2/1.3**（UDP 承载，[[mqtt]] 常用 MQTTS over TCP；CoAP+DTLS 才是它）：握手需 cookie 防放大、记录层加 epoch/seq 8 字节抗乱序重放。
- 设备身份：产线烧录客户端证书 → mTLS 一机一密；PSK 恢复省握手对电池设备友好。

---

## 🖼️ 10. TLS 记录层与握手时序图

![[TLS.excalidraw]]

---

## 🔗 思考与延伸索引

- [[http]] — HTTPS = HTTP over TLS；h2/h3 与 0-RTT 的交互
- [[SSH]] — 同为"传输+认证+通道"分层，但自成一派不用 X.509 链
- [[mqtt]] — MQTTS 端口 8883、PSK 套件与设备证书
- [[FTP]] — FTPS 显式 TLS（AUTH TLS）与数据连接也加密的坑
- [[TCP]]、[[UDP]] — TLS 之于 TCP；DTLS 之于 UDP 的重放窗口
- [[Socket]] — SSL_CTX 握一次、fd 复用的生命周期管理
- [[IP]] — SNI/ECH 与元数据可见性（中间盒识别）

---

## 📚 参考资料

1. **IETF** — *RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3*（记录层/握手/密钥推导/HKDF 全定义）.
2. **IETF** — *RFC 5246: TLS 1.2*（对照理解 PRF 与 legacy 握手）.
3. **IETF** — *RFC 9147: DTLS 1.3*；*RFC 6347: DTLS 1.2*（UDP 场景）.
4. **IETF** — *RFC 5869: HKDF*；*RFC 8448: TLS 1.3 Interop Test Vector*（自己逐步算一遍最治本）.
5. **IETF** — *RFC 8446 §4.2.11 与 RFC 9601: Encrypted Client Hello (ECH)*；*RFC 6962: Certificate Transparency*.
6. **Mozilla** — *Server-Side TLS (Security/Server-Side TLS) 推荐配置（modern/intermediate profile）*；**SSL Labs** — *SSL Server Test 评分方法论*.
7. **Arm** — *Mbed TLS 用户手册*（内存裁剪章节：MBEDTLS_SSL_IN_CONTENT_LEN / PSA 集成）；**wolfSSL** — *TLS 1.3 on Embedded Systems*.
8. **Nikita Borisov, Tianxiang Li 等论文/博客** — *High-Speed TLS 1.3*；Imperva/Hypothese 的 0-RTT 重放案例分析.
