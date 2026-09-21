---
title: SSH 协议完整技术手册
tags:
  - Network
  - Protocol
  - SSH
  - KEX
  - PortForwarding
status: Completed
created: 2026-09-21
---

# 🔗 SSH 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：Secure Shell（安全外壳）—— 运行在 **TCP 22** 之上的加密远程登录 / 命令执行 / 端口转发隧道协议。用一条认证加密的 TCP 连接替代 telnet/rlogin/rcp 等明文协议，并顺带成为 SFTP、SCP、rsync、git-over-ssh 的传输底座。当前版本 **SSH-2.0**（协议版本 2，与 SSH-1.x 报文格式完全不兼容，必须显式禁用）。
> - **内容范围**：三层架构 · 二进制报文帧格式 · 版本协商 · 密钥交换（KEX/DH group exchange + 主机密钥签名） · 用户认证 · channel 多路复用 · 三种端口转发 · HMAC/rekey · sshd 运维与安全加固
> - **推荐路径**：§1 分层定位 → §2 帧格式（协议的"物理层"）→ §3~§4 建连与密钥交换 → §5 认证 → §6~§7 连接层与转发 → §8~§9 完整性与运维 → §10 图 + 校验清单

---

## 🧭 1. 协议定位与三层架构

### 1.1 基本属性

| 属性 | 说明 |
| :--- | :--- |
| 定义 | SSH（Secure Shell Protocol），IETF SESHL 工作组标准化，服务器与客户端均为 C/S 角色 |
| 承载协议 | **TCP**，默认端口 **22**（可由 `Port` 改，但仅是噪声式防护，非安全边界） |
| 协议版本 | **SSH-2.0**（RFC 4251~4254）。`SSH-1.99` 表示"我同时听得懂 1.x"，SSH-1.x 已全面废弃 |
| 提供的能力 | 交互式 shell、单条命令执行（`exec`）、子系统（sftp）、本地/远程/动态端口转发、`ProxyJump` 跳板、隧道封装任意 TCP 服务 |
| 安全属性 | 服务器身份认证（主机密钥签名）→ 密钥交换（前向保密）→ 对称加密 + HMAC 完整性 → 用户认证 → 多路复用逻辑通道 |
| 与 TLS 的差别 | SSH 是**自成一体的二进制帧协议**，自带 KEX/认证/连接三层；TLS 只提供会话密钥与证书身份，不定义"通道"与"登录"语义 |
| 典型实现 | OpenSSH（`usr.bin/ssh`）、libssh、PuTTY、WinSCP、dropbear（嵌入式）、Go `golang.org/x/crypto/ssh` |

### 1.2 三层架构图

```text
┌────────────────────────────────────────────────────────────────┐
│  应用 / 子系统   shell · exec · sftp · scp -C · git-upload-pack │
├────────────────────────────────────────────────────────────────┤
│  ③ 连接层 Connection          RFC 4254                         │
│     channel 多路复用 / 窗口流控 / session / direct-tcpip ...     │
│     —— 一条 TCP 上跑 N 条互不干扰的双向"逻辑管道"                 │
├────────────────────────────────────────────────────────────────┤
│  ② 用户认证层 User Auth       RFC 4252                         │
│     publickey / password / keyboard-interactive / hostbased      │
│     —— 只回答"你能不能用这条已加密的通道"                        │
├────────────────────────────────────────────────────────────────┤
│  ① 传输层 Transport           RFC 4253                         │
│     二进制报文帧 · KEX（DH / curve25519）· 对称加密 · HMAC · 压缩  │
│     —— 建立机密性/完整性/服务器身份，产出 session_id              │
├────────────────────────────────────────────────────────────────┤
│  TCP 端口 22                                                  │
└────────────────────────────────────────────────────────────────┘
```

### 1.3 各层职责边界

| 层 | 负责 | 明确不负责 |
| :--- | :--- | :--- |
| **传输层 Transport** | 报文分帧、算法协商、密钥交换、服务器身份验证、加密与 MAC、压缩、`rekey`、`service-name` 声明 | 不知道"用户是谁"，不知道"第 3 条 channel 是什么" |
| **认证层 User Authentication** | 认证方法协商（`none` 探测可用方法）、公钥/口令/键盘交互认证、认证成功后才允许开 channel | 不定义具体业务报文，仅在传输层之上复用帧格式 |
| **连接层 Connection** | channel 的建立/关闭、每 channel 窗口流控、`session` 通道内的 `pty-req/shell/exec/env/signal`、TCP/IP 转发报文 | 不改变加密与密钥，不感知认证细节 |

> [!NOTE] 为什么分层重要
> 排障时先定位层次：连不上/协商失败 → 传输层（KEX 算法不匹配、主机密钥变更）；`Permission denied (publickey)` → 认证层；能登录但转发不生效 → 连接层与 `sshd_config` 的 `AllowTcpForwarding`/`GatewayPorts`。

---

## 📦 2. 二进制报文协议（帧格式核心）

SSH 不是文本协议。除最初的 identification 行外，**所有报文都是同一种二进制包**（RFC 4253 §6）：

### 2.1 报文结构（binary packet）

```text
 uint32         uint8        variable length       variable      fixed length
┌───────────┬────────────┬─────────────────────┬─────────────┬────────────────┐
│ packet_   │ padding_   │ payload             │ padding     │ MAC            │
│ length    │ length     │ (byte packet_type +  │ (4~16 B     │ (HMAC / AEAD   │
│           │            │  若干字段)           │  随机字节)   │  tag)          │
└───────────┴────────────┴─────────────────────┴─────────────┴────────────────┘
     4 B          1 B           packet_length - 1 B        可变        算法定长
```

| 字段 | 类型 | 长度 | 约束与说明 |
| :--- | :--- | :--: | :--- |
| **packet_length** | `uint32`（网络字节序，大端） | 4 B | 等于 $1 + \text{payload} + \text{padding}$，**不含自身 4 B，也不含 MAC** |
| **padding_length** | `uint8` | 1 B | 后面 `padding` 的字节数 |
| **payload** | 变长 | ≥ 1 B | 首字节是 `packet_type`（报文编号，1~159），其余是该报文的参数 |
| **padding** | 变长 | ≥ 4 B，≤ 255 B | 随机填充，用于把包凑成分组长度的倍数（至少 8 B），并隐藏真实载荷长度 |
| **MAC** | 定长 | 16/32/64 B | 消息认证码，覆盖 $4 + \text{packet\_length}$ 字节；AEAD 模式合并为 tag |

**长度约束**：

$$
\text{packet\_length} = \text{padding\_length} + |\text{payload}| + |\text{padding}| \ge 12
$$

$$
(4 + \text{packet\_length}) \bmod \max(8,\ \text{block\_size}) = 0
$$

### 2.2 两种加密视图

| 视图 | 明文部分 | 加密部分 | MAC 输入 | 出现时机 |
| :--- | :--- | :--- | :--- | :--- |
| **A. 会话密钥建立前** | 整帧（`cipher = none`、`MAC = none`） | 无 | 不计算 | identification 行、`SSH_MSG_KEXINIT`、DH 交换阶段 |
| **B. 传统 CBC（"前 5 字节不加密"）** | `packet_length`(4 B) + `padding_length`(1 B) | `payload` + `padding` 按分组加密 | 长度字段 + 密文 | 早期 CBC 套件的经典描述 |
| **C. 整体加密（CTR / GCM / chacha20-poly1305）** | 仅 `packet_length`（GCM/ChaCha 用独立密钥把它也加密） | **整包**，长度字段亦进密码流 | ETM 变体：先加密再对密文算 MAC，AEAD 无独立 MAC 字段 | `SSH_MSG_NEWKEYS` 之后的常态 |

> [!IMPORTANT] 视图 A 与 C 的区别不是"细节"，而是攻击面
> 长度字段明文会泄露包长分布（流量分析）；ETM（Encrypt-then-MAC）先验 MAC 再解密，避免 CBC padding oracle 一类"解密前即拒绝"的路径，`hmac-*-etm@openssh.com` 就是为此存在。

### 2.3 最大包长与压缩

- **单包上限 350 KB**：`packet_length + 4`（含 MAC 前的整包）**不得超过 $350 \times 1024 = 358400$ 字节**；接收方发现超限必须以 `SSH_MSG_DISCONNECT(SSH_DISCONNECT_PROTOCOL_ERROR)` 断链。
- 上限存在的意义：给接收端一个可预测的静态缓冲上界（这也是 dropbear 等嵌入式实现能零动态分配的前提）。
- **channel 层的最大报文**另有限制：`CHANNEL_OPEN` 携带 `maximum packet size`（OpenSSH 常见 32768 B），大数据必须切多个 `CHANNEL_DATA`。
- **压缩**在加密之前（先压缩后加密，接收端解密后解压）：
  - `none` —— 默认，避免压缩侧信道；
  - `zlib` —— RFC 4253 定义，握手期即压缩；
  - `zlib@openssh.com` —— OpenSSH 扩展，**认证完成后才启用压缩**，规避对 KEX/认证明文的 CRIME/BREACH 类压缩泄露。

### 2.4 顶层 payload 报文编号速查

| 编号区间 | 层 | 代表报文 |
| :--: | :--- | :--- |
| 0 | — | `SSH_MSG_IGNORE` |
| 1~2 | 传输 | `DISCONNECT`、`IGNORE` |
| 3~4 | 传输 | `UNIMPLEMENTED`、`DEBUG` |
| 5~7 | 传输 | `SERVICE_REQUEST`、`SERVICE_ACCEPT` |
| 20~22 | 传输 | `EXT_INFO` 等（RFC 8308） |
| 30~49 | 密钥交换 | `KEXINIT`(30/31)、`NEWKEYS`(21)、`KEXDH_INIT`(30)/`KEXDH_REPLY`(31)、GEX 系列 31~34 |
| 50~59 | 认证 | `USERAUTH_REQUEST`(50)、`FAILURE`(51)、`SUCCESS`(52)、`BANNER`(53)、`INFO_REQUEST/RESPONSE`(54/55) |
| 60~79 | 连接 | `GLOBAL_REQUEST`(80)… channel 相关：`OPEN`(90)、`OPEN_CONFIRMATION`(91)、`OPEN_FAILURE`(92)、`WINDOW_ADJUST`(93)、`MAX_PACKET`(94)、`CLOSE`(96)、`REQUEST`(98)、`SUCCESS`(99)、`DATA`(94/95)… |

> [!WARNING] 编号易错点
> `SSH_MSG_NEWKEYS` 是 **21**，紧跟在 KEX 系列之后；`CHANNEL_DATA`(94)/`CHANNEL_WINDOW_ADJUST`(93) 与 transport 层的报文段不同区间。真正写解析器时应以 RFC 4250/4251 的编号注册表为准，不要凭记忆。

---

## 📜 3. 版本协商（Identification String）

连接建立后、任何二进制包之前，双方各发一行 ASCII banner：

```text
SSH-protoversion-softwareversion SP comments CR LF
```

实例：

```text
SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.5
SSH-2.0-libssh_0.10.6
SSH-1.99-OpenSSH_8.2                 ← 声称向下兼容 SSH-1.x
```

规则（RFC 4252/4253 §4.2）：

1. 以 `SSH-` 开头，第一个 `-` 之后是协议版本（`1.99` 或 `2.0`），第二个 `-` 之后是软件名与版本；
2 **banner 行不加密、不压缩、不参与 MAC**；每行以 `CR LF` 结束；
3. 版本行之前允许插入任意多行注释（不得包含 `-`），可用于打印法律告示；
4. **必须收到对方 identification 后才能发送 `SSH_MSG_KEXINIT`**；
5. 客户端常用探测：`nc host 22` 或 `ssh -G`/`ssh -vvv` 观察 banner，判断厂商与版本（同时是资产指纹泄露点）。

---

## 🔑 4. 密钥交换（Key Exchange, KEX）

### 4.1 算法协商：SSH_MSG_KEXINIT

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `cookie` | 16 B 随机 | 唯一标识本次 KEX，防止报文被复用/伪造 |
| `kex algorithms` | 名称列表 | `curve25519-sha256`、`ecdh-sha2-nistp256`、`diffie-hellman-group16-sha512`、`diffie-hellman-group-exchange-sha256` |
| `server host key algorithms` | 名称列表 | `ssh-ed25519`、`rsa-sha2-512`、`ecdsa-sha2-nistp384`、`ssh-rsa`（SHA-1，已淘汰） |
| `encryption_algorithms` c→s / s→c | 名称列表 | `chacha20-poly1305@openssh.com`、`aes256-gcm@openssh.com`、`aes256-ctr` |
| `mac_algorithms` c→s / s→c | 名称列表 | `hmac-sha2-256-etm@openssh.com`、`umac-128-etm@openssh.com` |
| `compression_algorithms` c→s / s→c | 名称列表 | `none`、`zlib@openssh.com` |
| `languages` c→s / s→c | 名称列表 | 通常为 `""` |
| `first_kex_packet_follows` | boolean | 猜测加速：紧跟一个 KEX 报文 |
| `0` | uint32 | 保留，必须为 0 |

协商算法 = **客户端列表优先**，按顺序在服务器列表中寻找第一个同名项（各方向独立）。无交集时服务器回 `SSH_MSG_DISCONNECT(KEY_EXCHANGE_METHOD_NOT_SUPPORTED)`。

### 4.2 Diffie-Hellman Group Exchange 流程

```text
Client                                            Server
  │ ── SSH_MSG_KEX_DH_GEX_REQUEST(min=1024,n=3072,max=8192) ─▶ │   ① 我要多大的群
  │ ◀── SSH_MSG_KEX_DH_GEX_GROUP(p, g) ────────────────────── │   ② 安全素数群 p=2q+1
  │ ── SSH_MSG_KEX_DH_GEX_INIT(e = g^x mod p) ──────────────▶ │   ③ 我的临时公钥
  │ ◀── SSH_MSG_KEX_DH_GEX_REPLY(f, K_S, sig) ─────────────── │   ④ 你的公钥 + 主机公钥 + 签名
  │ ── SSH_MSG_NEWKEYS ─────────────────────────────────────▶ │   ⑤ 切换密钥
  │ ◀── SSH_MSG_NEWKEYS ───────────────────────────────────── │   ⑥ 切换密钥
```

共享秘密与会话哈希（MPINT = 4 字节长度 + 大端字节串，负数补 0 字节）：

$$
K \;=\; f^{\,x} \bmod p \;=\; g^{\,xy} \bmod p
$$

$$
H \;=\; \text{HASH}\big(V_C \,\|\, V_S \,\|\, I_C \,\|\, I_S \,\|\, K_S \,\|\, \text{min} \,\|\, n \,\|\, \text{max} \,\|\, p \,\|\, g \,\|\, e \,\|\, f \,\|\, K\big)
$$

其中 $V_C/V_S$ 为双方 identification 版本行，$I_C/I_S$ 为完整 `SSH_MSG_KEXINIT` 报文。$H$ 把"所有协商结果"绑定进签名，任何算法降级都会被检测（防比特翻转/降级攻击）。

会话密钥按字符标签派生（RFC 4253 §7.2）：

$$
K_1 = \text{HASH}(K \,\|\, H \,\|\, \texttt{'A'} \,\|\, \text{session\_id}),\quad
\text{IV}_{c\to s}(\texttt{C}),\ \text{IV}_{s\to c}(\texttt{D}),\ \text{MAC}_c(\texttt{E}),\ \text{MAC}_s(\texttt{F})
$$

- **DH 群参数**：GEX 的 modp 群取自 RFC 3526 / RFC 4419（2048/3072/4096/6144/8192 bit）；固定群版本为 `diffie-hellman-group14-sha256` 等。
- **前向保密（PFS）**：$x,y$ 是临时私钥，`rekey` 后旧密钥无法还原历史流量 —— 这是 `curve25519-sha256`/ECDH 取代静态 RSA 传输层密钥交换的核心理由。

### 4.3 服务器身份与主机密钥签名（known_hosts 信任模型）

1. 服务器用**主机私钥**（`/etc/ssh/ssh_host_ed25519_key`）对 $H$ 签名：`sig = Sign(host_privkey, H)`；
2. 客户端在 `SSH_MSG_KEX_DH_GEX_REPLY` 中同时收到主机公钥 $K_S$ 与签名；
3. 客户端先在 `~/.ssh/known_hosts` 查该主机（按 `[host]:port` 或主机名匹配）中记录的 $K_S$：
   - **无记录** → 打印公钥指纹（SHA256 base64），要求用户确认（**Trust On First Use, TOFU**），接受则追加写入；
   - **有记录且验签通过** → 服务器身份成立，继续认证；
   - **记录与收到的 $K_S$ 不一致** → 报 `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`，拒连（`ssh-keygen -R <host>` 清理旧条目，需先确认是合法换机还是 MITM）；
   - **无记录但配了 `StrictHostKeyChecking=yes`**（如 CI/镜像） → 直接失败，不询问。
4. `session_id` = **第一次** KEX 的 $H$，后续 `rekey` 不更新，用于把认证签名钉死在"最初那次密钥交换"上，防止跨会话重放。

| known_hosts 形态 | 示例 | 说明 |
| :--- | :--- | :--- |
| 明文 | `git.example.com ssh-ed25519 AAAAC3Nza...` | 单主机名单行 |
| 带端口 | `[10.0.0.5]:2222 ssh-ed25519 AAAA...` | 非 22 端口需方括号 |
| 通配/多主名 | `*.corp.example.com,10.0.0.* ssh-rsa AAAA...` | 谨慎使用 |
| 已哈希 | `\|1\|J7m1...= \|1\|kQ2x...= ssh-ed25519 ...` | `HashKnownHosts yes`，防局域网主机名被枚举 |
| 证书 | `@cert-authority *.example.com ssh-ed25519 ...` | 用 CA 签发的 `ssh-ed25519-cert-v01@openssh.com`，替代 TOFU |

> [!TIP] 批量预置主机密钥
> `ssh-keyscan -t ed25519 host > ~/.ssh/known_hosts` 只抓取、**不验证**。生产应在带外渠道核对指纹，或改走 SSH 证书（`sshd` 配 `TrustedUserCAKeys` + 客户端 `CanonicalDomains`），彻底摆脱 TOFU 的人工环节。

---

## 🪪 5. 用户认证层

### 5.1 认证方式对照

| 方法名 | 原理 | 安全性 | 典型场景 |
| :--- | :--- | :---: | :--- |
| **publickey** | 服务器用 `authorized_keys` 中的公钥验证客户端对会话上下文的签名；**私钥永不出本机** | 高（推荐） | 运维/CI/Git，默认唯一应启用的方式 |
| **password** | 口令经已加密通道传给服务器 | 中（受撞库/钓鱼威胁） | 首台机器初始化、无 agent 环境 |
| **keyboard-interactive** | 服务器逐条下发 prompt（RFC 4256），客户端回 N 条响应 | 中高 | PAM 双因素（TOTP/短信）、密码过期改密 |
| **hostbased** | 用"客户端主机的 host key + `shosts.equiv`/`.shosts`"证明来源主机身份 | 低（信任面过大） | 内网互信历史方案，现代禁用 |
| **none** | 空方法，仅用于查询 `allowed methods` | — | 探测/调试 |

### 5.2 publickey 原理（两阶段设计）

```text
阶段一（探测，不签名，省 CPU / 支持硬件 key 预检）
  C ─▶ S : SSH_MSG_USERAUTH_REQUEST(user, "ssh-connection", "publickey",
                                    has_signature = FALSE, algo, public_key_blob)
  S ─▶ C : SSH_MSG_USERAUTH_FAILURE(allowed = "publickey,password", partial = FALSE)
             ↑ 若公钥在 authorized_keys 中且允许，则回 "pk-ok"（OpenSSH 走带签名的快路径）

阶段二（真签名）
  C ─▶ S : SSH_MSG_USERAUTH_REQUEST(..., "publickey", has_signature = TRUE,
                                    algo, blob, signature = Sign(privkey,
                                            session_id ‖ 认证请求字段 ‖ blob))
  S      : 查 ~/.ssh/authorized_keys → 用该公钥验签 → 比对算法与 key 类型
  S ─▶ C : SSH_MSG_USERAUTH_SUCCESS / FAILURE
```

- **签名内容含 `session_id`**：把这次认证绑定到本次 KEX 的哈希上，中间人无法把签名搬到自己与服务器建立的会话里重放。
- **私钥从不传输**，网络上只出现公钥 blob 与签名 —— 这正是 publickey 优于 password 的本质（服务器被攻破/流量被镜像都拿不到口令）。
- 若私钥带口令（passphrase），签名前需本地解密私钥，可由 `ssh-agent` 托管，避免每次输入。

### 5.3 密钥生成与部署（OpenSSH 常用命令）

| 命令 | 作用 |
| :--- | :--- |
| `ssh-keygen -t ed25519 -a 100 -C "ops@laptop" -f ~/.ssh/id_ops_ed25519` | 生成 Ed25519 密钥（推荐），KDF 100 轮，带注释 |
| `ssh-keygen -t rsa -b 4096 -o -C ...` | 旧系统只吃 RSA 时用 4096 bit + `-o`（PKCS#8/OpenSSH 新格式） |
| `ssh-keygen -p -f id_ops_ed25519 -a 100` | 修改/增删私钥口令（改 passphrase，不改变密钥） |
| `ssh-keygen -y -f id_ops_ed25519 > id_ops_ed25519.pub` | 从私钥重新导出公钥 |
| `ssh-keygen -l -f id_ops_ed25519.pub` | 打印指纹（SHA256:...）与位数 |
| `ssh-keygen -R 10.0.0.5` / `-H` / `-F host` | 删除主机条目 / 哈希整个文件 / 查询是否已知 |
| `ssh-copy-id -i ~/.ssh/id_ops_ed25519.pub -p 22 ops@host` | 追加公钥到远端 `~/.ssh/authorized_keys`（自动建目录并设权限） |
| `eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_ops_ed25519` | agent 托管私钥；Windows 用 OpenSSH Authentication Agent 服务 |
| `ssh -o PreferredAuthentications=publickey -o IdentitiesOnly=yes -i key host` | 只用指定 key，防止 agent 里所有 key 被服务器逐个试出 |

PowerShell / Windows 平台注意：`%USERPROFILE%\.ssh` 权限用 `icacls` 收紧；`authorized_keys` 若位于管理员账户，实际生效路径是 `C:\ProgramData\ssh\administrators_authorized_keys`，且属主与权限必须为 SYSTEM/Administrators 独占。

### 5.4 authorized_keys 条目格式

```text
restrict,command="rsync --server -eglqt.prss --sender .",no-port-forwarding,no-agent-forwarding ssh-ed25519 AAAAC3Nza... deploy@git
│└─ 选项（逗号分隔）                                    │  └─ base64 公钥            └─ 注释
└─ 密钥类型
```

常用选项：`from="10.0.0.0/24"`（源地址限制）、`command="..."`（强制命令，Git 托管核心）、`no-pty`、`no-x11-forwarding`、`environment="KEY=v"`、`ttl` 需配合证书。

---

## 🔀 6. 连接层：channel 多路复用

### 6.1 channel 模型

- 一条 TCP 连接上并行多条 **channel**，每条由两端各自分配的 `uint32 channel_id` 标识：`sender channel`（发起方本地编号）与 `recipient channel`（对方编号），`OPEN_CONFIRMATION` 完成映射表绑定；
- 所有 channel 共用同一加密流，交错传输靠 `recipient channel` 解复用 —— 这是"一条 SSH 连接同时跑 shell + sftp + 若干转发"的机制；
- **每条 channel 一个半关闭状态机**：`CHANNEL_CLOSE` 只关自己，不影响同连接其他 channel；全部关闭后可复用同一 TCP（配合 `ControlMaster`）；
- **流控靠窗口**：接收方通告 `initial window size` + `maximum packet size`；数据每消耗窗口发 `CHANNEL_WINDOW_ADJUST`，发送方窗口为 0 时必须停发。窗口不重置（`rekey` 后不清零），是 SSH 与 HTTP/2 类协议的重要差异。

```text
Client channel 表                       Server channel 表
  local_id ─┐   ┌─ remote_id              local_id ─┐   ┌─ remote_id
  0  ───────┴───┼──► 17                    17 ───────┴───┼──► 0
  1  ───────────┼──► 18   （session/转发混跑） 18 ───────────┼──  1
```

### 6.2 channel 类型表

| 类型名 | 方向/语义 | 附加字段 | 用途 |
| :--- | :--- | :--- | :--- |
| **session** | C→S 打开 | — | 交互式 shell、`exec` 单命令、`subsystem sftp`，认证层通过后最常见 |
| **direct-tcpip** | C→S 打开 | `host to connect`, `port`, `originator ip/port` | 承载 `-L` 本地转发：客户端把连到本地端口的字节流送进来，由服务器外连目标 |
| **forwarded-tcpip** | S→C 打开 | 同上（服务器侧为发起者） | 承载 `-R` 远程转发：远端监听端口被连接后，服务器反向开 channel 推给客户端 |
| **x11-connection** | S→C 打开 | originator 填 `"x11"` | X11 Forwarding（`ForwardX11 yes`）：伪造的 `DISPLAY:10.0` 回连客户端 X Server |
| **auth-agent-connection@openssh.com** | S→C 打开 | — | Agent Forwarding：远端程序经 socket 借用本地 agent 签名 |
| **tun / tap** | 双向 | tunnel mode | `ssh -w` 建立 L2/L3 隧道，配合 `AllowTun/TapForwarding` 组网 |
| **unix-socket / unix-stream@openssh.com / forward-socket@openssh.com** | 双向 | socket path | 本地 Unix socket 转发（`-L /tmp/x.sock`） |

### 6.3 session channel 内的 CHANNEL_REQUEST

| request 类型 | 作用 | 对应命令 |
| :--- | :--- | :--- |
| `pty-req` | 申请伪终端（`TERM`、窗口行列、终端模式） | 登录默认；`ssh -T` 跳过 |
| `shell` | 启动用户的登录 shell | `ssh host` |
| `exec` | 执行单条命令后（可能）关闭 | `ssh host df -h` |
| `subsystem` | 启动子系统（`sftp`） | `sftp`、`scp -O`、VS Code Remote-SSH |
| `env` | 传环境变量（受 `AcceptEnv` 限制） | `ssh -o 'SetEnv FOO=bar'` / `SendEnv` |
| `window-change` | 终端 resize 通知 | 窗口大小变化 |
| `signal` / `xon-xoff` | 发信号 / 流控状态 | `Ctrl-C` 映射 |

> [!NOTE] exit-status 与 exit-signal
> 远端命令结束后服务器发 `CHANNEL_REQUEST(exit-status)`（类型串见 RFC 4254 §6.10），客户端 `ssh` 据此设置自身退出码 —— 脚本里 `ssh host cmd && ...` 的返回值语义来自这里，而不是 TCP 关闭。

---

## 🚪 7. 端口转发三模式

### 7.1 本地转发 `-L`（Local，最常用）

```bash
ssh -N -L 127.0.0.1:5432:db.internal:5432 ops@bastion.example.com
```

```text
你的 App ──TCP──▶ 本地 127.0.0.1:5432 (ssh 监听)
                       │ 封装进 channel: direct-tcpip
                       ▼
                 SSH 隧道（加密）──▶ 堡垒机 sshd
                                       │ 由 sshd 外连
                                       ▼
                                   db.internal:5432
```
流向：**本地端口 → channel(`direct-tcpip`) → 服务器侧 → 目标**。目标地址是"从服务器看"的可达地址。

### 7.2 远程转发 `-R`（Remote）

```bash
ssh -N -R 0.0.0.0:8080:localhost:3000 ops@bastion.example.com
```

```text
外部访问者 ──TCP──▶ 服务器 0.0.0.0:8080 (sshd 监听)
                        │ 反向开 channel: forwarded-tcpip
                        ▼
                  SSH 隧道（加密）──▶ 本地 ssh 客户端
                                        │ 本地外连
                                        ▼
                                    localhost:3000 (dev server)
```
流向：**远端监听端口 → channel(`forwarded-tcpip`) → 客户端 → 客户端本地的目标**。常用于内网穿透 / 演示回调，需 `sshd_config` 允许 `AllowTcpForwarding yes`，非回环绑定还需 `GatewayPorts yes|clientspecified`。

### 7.3 动态转发 `-D`（Dynamic SOCKS 代理）

```bash
browser --proxy-server=socks5://127.0.0.1:1080
ssh -N -D 1080 ops@bastion.example.com
```

```text
浏览器 ──SOCKS5 握手(域名/地址)──▶ 本地 1080 (ssh 内置 SOCKS 服务)
                                       │ 按 SOCKS 请求里的目标地址
                                       ▼ 开 channel: direct-tcpip
                                 堡垒机 sshd ──TCP──▶ 任意目标:80/443/...
```
流向：**本地 SOCKS5 → 每个新目标动态开一条 channel → 服务器去连**。区别：`-L` 目标固定写死在命令行，`-D` 目标由 SOCKS 客户端在运行时告知，因此"一条 `-D` 顶 N 条 `-L`"，但无法给不支持 SOCKS 的老程序用（可用 `proxychains` 兜底）。

### 7.4 对照与配套项

| 项 | `-L` | `-R` | `-D` |
| :--- | :---: | :---: | :---: |
| 监听位置 | 客户端本地 | 服务器端 | 客户端本地 |
| 目标决定时机 | 命令行（固定） | 命令行（固定） | 运行时（SOCKS 请求） |
| channel 类型 | `direct-tcpip` | `forwarded-tcpip` | `direct-tcpip` |
| 服务器配置 | `AllowTcpForwarding`、`PermitOpen` | `AllowTcpForwarding`、`GatewayPorts`、`PermitListen` | `AllowTcpForwarding` |
| 常用配套 | `-N`（不开 shell）、`-f`（后台）、`-o ServerAliveInterval=30`、`-o ExitOnForwardFailure=yes` | 同左 | 浏览器/systemd 环境注入 |

> [!TIP] 转发排障三步
> 1. `ss -lntp` / `netstat -ano` 确认监听端口与地址族（`127.0.0.1` vs `0.0.0.0`）；
> 2. `ssh -vvv` 看有没有出现 `channel 2: open ... direct-tcpip` 与 `open failed: administratively prohibited`（→ 服务器禁转发）；
> 3. 加 `-o ExitOnForwardFailure=yes`，让"端口没起来"直接失败而不是静默半连接。

---

## 🛡️ 8. 完整性、rekey 与 sshd 运维

### 8.1 报文完整性（HMAC）

$$
\text{MAC} \;=\; \text{HMAC}_{K_{mac}}\big(\text{seq\_nr} \,\|\, \text{packet\_length} \,\|\, \text{padding\_length} \,\|\, \text{payload} \,\|\, \text{padding}\big)
$$

- 密钥双向独立（`E`/`F` 派生），算法 128/256 bit 输出（`hmac-sha2-256`、`umac-64`）；
- **ETM 变体**（`*-etm@openssh.com`）：对**密文**计算 MAC，接收端先验 MAC 再解密，杜绝 padding oracle；
- **AEAD 套件**（`aes128-gcm@openssh.com`、`chacha20-poly1305@openssh.com`）：MAC 字段被合并进认证 tag，ChaCha20 变体另用第二把密钥单独加密 `packet_length`；
- **报文序号 `seq_nr` 参与 MAC 且从不重置**（跨 `rekey` 延续），专门用来防截断/重放攻击（RFC 4344 的 truncation attack 分析）；32 bit 回绕时 RFC 要求以 `rekey` 处理。

### 8.2 重协商（rekey）触发条件

| 触发源 | 默认/典型值 | 说明 |
| :--- | :--- | :--- |
| 数据量 | OpenSSH 1 GiB（`RekeyLimit default 1h`） | 对称密钥用量到达密码学安全上界；对 ChaCha20 更重要（keystream 复用风险） |
| 时间 | 1 小时 | 与数据量并列，谁先到谁生效 |
| 手动 | 客户端 `SSH_MSG_KEXINIT` 或运维信号 | `ssh -o 'ServerAlive...'` 无关；用于强制换密钥 |
| 算法/扩展到期 | `EXT_INFO` / 服务端策略 | 例如强制迁移到 ETM/AEAD |
| 序号回绕 | `seq_nr` 接近 $2^{32}$ | 必须 `rekey`（RFC 4344） |

`rekey` 期间双方挂起其他报文交换（"KEX 阶段阻塞"），这就是长时间大流量传输偶发抖动的原因；`diffie-hellman-group-exchange` 可协商**更大的群**，而静态群版本不行。

### 8.3 sshd_config 关键项

| 配置项 | 推荐值 | 作用与风险 |
| :--- | :--- | :--- |
| `Port` | 22（或自定义） | 只降噪声日志量，**不是安全控制**；需 SELinux/firewall 同步 |
| `ListenAddress` | 明确内网 IP | 多网卡时避免暴露到公网接口 |
| `PermitRootLogin` | `prohibit-password`（默认）/ `no` | `yes` 允许 root 口令登录，最高危；`no` 会同时禁止 root 密钥登录（备份账号绕行） |
| `PasswordAuthentication` | `no` | 关密码，仅密钥；防撞库/爆破 |
| `KbdInteractiveAuthentication` | `no`（除非接 2FA/PAM） | 常被误认为"已禁用密码"，实际是另一条通道 |
| `PubkeyAuthentication` | `yes` | 主认证路径 |
| `AuthenticationMethods` | `publickey keyboard-interactive:pam` | 多因素串联写法 |
| `MaxAuthTries` / `LoginGraceTime` | 3~6 / 30 | 限制单连接认证尝试与未认证占坑时长 |
| `MaxStartups` | `10:30:60` | 未认证并发连接限速，抗握手洪水 |
| `AllowUsers` / `AllowGroups` | 白名单 | 比 `DenyUsers` 更安全的默认拒绝 |
| `AllowTcpForwarding` / `PermitOpen` | 按需 `local` / 显式 `host:port` | 转发是横向移动通道，最小化 |
| `GatewayPorts` | `no` | `yes` 会让 `-R` 绑到 `0.0.0.0`，暴露面陡增 |
| `X11Forwarding` | `no`（除非必要） | 开启即引入 X 信任链 |
| `AllowAgentForwarding` | `no`（生产） | agent 转发 = 私钥可被同机 root 借用 |
| `Ciphers` / `MACs` / `KexAlgorithms` / `HostKeyAlgorithms` | 显式白名单（`chacha20-poly1305@openssh.com,aes256-gcm@openssh.com` / `hmac-sha2-256-etm@openssh.com` / `curve25519-sha256` / `ssh-ed25519,rsa-sha2-512`） | 排除 3DES、Blowfish、`ssh-rsa`(SHA-1)、`group1/group14-sha1` |
| `ClientAliveInterval` / `ClientAliveCountMax` | 60 / 3 | 清理 NAT 后失效的空闲会话 |
| `StrictModes` | `yes` | 目录/文件权限不合格即拒绝登录（见下） |
| `UsePAM` / `Banner` / `LogLevel` | `yes` / `/etc/issue.net` / `VERBOSE` | 合规告示与审计粒度 |

### 8.4 权限铁律（StrictModes 会检查）

| 对象 | 权限 | 属主要求 |
| :--- | :--: | :--- |
| `~/.ssh` 目录 | `700` | 用户本人，且父目录不可被他人写 |
| `~/.ssh/authorized_keys`、`known_hosts`、`config` | `600`（`config` 可 `644`） | 用户本人 |
| 私钥 `id_ed25519` | `600`（组/其他必须无权限，否则 `ssh` 直接拒绝使用） | 用户本人 |
| 主机私钥 `/etc/ssh/ssh_host_*_key` | `600`，属 `root` | 公开证书 `*.pub` 644 |
| 远端 home 目录 | 不允许 group/world 可写 | 否则 `StrictModes yes` 拒登录 |

客户端配置示例：

```sshconfig
# ~/.ssh/config
Host bastion
    HostName 203.0.113.10
    User ops
    IdentityFile ~/.ssh/id_ops_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 30
    ServerAliveCountMax 3

Host db-*
    User dbops
    ProxyJump bastion          # 跳板机
    ForwardAgent no
```

---

## 🔒 9. 与安全实践的衔接

### 9.1 禁用密码登录 + 失败封禁（fail2ban）

```ini
# /etc/ssh/sshd_config
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
MaxAuthTries 3
LoginGraceTime 30

# /etc/fail2ban/jail.d/sshd.local
[sshd]
enabled = true
port    = 22
maxretry = 5
findtime = 10m
bantime  = 1h        # 递进可用 bantime.increment = true
backend  = systemd   # 现代发行版直接读 journald，免依赖 auth.log
```

配合 `sssd`/PAM `pam_faillock` 做账户级锁定；`bantime` 过短对分布式慢速爆破无效，过长会误封自家跳板 —— 用 `ignoreip` 白名单固定运维出口。

### 9.2 跳板机与 ProxyJump

```bash
ssh -J bastion,internal-gw db01                     # 两级跳板
ssh -o ProxyJump=ops@bastion -L 5432:db:5432 db01   # 跳板 + 转发
ssh -W internal-gw:22 ops@bastion                   # ProxyJump 的底层实现
```

- `-J` 的每一跳都要**独立开启转发许可**（中间节点 `AllowTcpForwarding yes`），否则表现为 `open failed: administratively prohibited`；
- `ProxyCommand ssh -W %h:%p bastion` 是等价老写法，可塞进任意命令（`nc`、`corkscrew`、`warp`）；
- 跳板机应：只允许密钥、只允许 `netcat` 式转发、禁 shell（`command="..."` 强制）或走 SFTP-only + 会话审计（`script`/`LogVol`）；堡垒机日志按 `session_id` 归档，才能证明"谁在何时操作了哪台"。

### 9.3 其余必做项

| 方向 | 做法 |
| :--- | :--- |
| 密钥生命周期 | 定期轮换 `authorized_keys`；生产改 SSH 证书（短 `Valid Principals` + `ttl`），避免"公钥散布在 300 台机器" |
| 算法治理 | 禁用 `ssh-rsa`(SHA-1)、`diffie-hellman-group1-sha1`、`arcfour`、`3des-cbc`、`hmac-md5/96`；用 `ssh -Q kex/cipher/mac` 与 `nmap --script ssh2-enum-algos` 自检 |
| 版本与暴露面 | 及时打 OpenSSH 补丁（CVE 高频区）；公网只暴露跳板，DB/管理端口靠隧道而非 `0.0.0.0` 监听 |
| 检测 | 记录 `sshd` 认证事件（成功/失败/来源）入 SIEM；`known_hosts` 集中管控；对 `Host key mismatch` 告警当作安全事件处理 |
| 常见坑 | `ControlMaster` 复用旧会话会掩盖密码/密钥失效；`ssh -N` 忘加导致意外开 shell；`-R` 端口被 `GatewayPorts no` 静默绑回环；agent 被 `AllowAgentForwarding` 借走私钥签名能力 |

---

## 🖼️ SSH 分层握手时序与报文结构图

![[SSH.excalidraw]]

- 左画布：二进制报文结构 `packet_length / padding_length / payload / padding / MAC` + "会话密钥建立前 MAC=none" 与 "协商加密后整包加密" 两种视图对比；
- 右画布：TCP 连接 → identification banner 互换 → `KEXINIT` 算法协商 → DH GEX 交换与主机密钥签名 → `NEWKEYS` → `userauth_request(publickey)` 挑战签名 → `SUCCESS` → `CHANNEL_OPEN(session)` → `shell` 数据双向 → 关闭。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] SSH 服务/客户端上线前必过
> - [ ] **协议版本**：仅接受 `SSH-2.0-*`，`Protocol 2` 语义下 1.x 客户端一律拒绝
> - [ ] **算法白名单**：KEX 用 `curve25519-sha256`/GEX-SHA256；主机密钥 `ssh-ed25519`/`rsa-sha2-512`；加密用 AEAD 或 CTR；MAC 用 `*-etm@openssh.com`；禁用 `ssh-rsa`(SHA-1)、`group1*`、`3des`、`arcfour`
> - [ ] **主机密钥信任**：首次指纹经带外核对；`known_hosts` 已集中分发或改走 SSH 证书；对 `REMOTE HOST IDENTIFICATION HAS CHANGED` 有响应流程
> - [ ] **认证**：`PasswordAuthentication no` + `KbdInteractiveAuthentication no`；`PermitRootLogin no|prohibit-password`；`AllowUsers/AllowGroups` 白名单；`MaxAuthTries` 收敛
> - [ ] **密钥与权限**：私钥/`authorized_keys` `600`、`~/.ssh` `700`、home 不可 group 写；`StrictModes yes` 未被关成 `no`
> - [ ] **转发策略**：`AllowTcpForwarding`/`PermitOpen`/`PermitListen` 最小化；`GatewayPorts no`；`AllowAgentForwarding no`；X11 按需
> - [ ] **完整性与会话**：`RekeyLimit` 显式（默认 1 GiB/1 h 可接受）；确认 `hmac-*-etm` 生效；ETM/AEAD 优先
> - [ ] **可用性**：`ClientAliveInterval/CountMax` 清理僵尸会话；`MaxStartups` 限速；`Banner` 合规告示；`LogLevel VERBOSE` 保留公钥指纹日志
> - [ ] **防爆破**：fail2ban/sshguard + `ignoreip` 白名单 + 递进 `bantime`；日志接入 SIEM
> - [ ] **跳板链路**：每一跳 `AllowTcpForwarding` 已开、`ProxyJump` 多跳可连通、审计按会话留痕
> - [ ] **客户端**：`IdentitiesOnly yes` 防 agent 广播所有 key；`-o ExitOnForwardFailure=yes` 让转发失败可见；脚本 `ssh host cmd` 依赖 `exit-status` 判定成败

---

## 🔗 思考与延伸索引

- [[TCP]] —— SSH 的分帧与"粘包"问题本质是 TCP 字节流之上的报文定界；`packet_length` 即自描述定界方案；rekey 期间的阻塞与 TCP 重传叠加造成卡顿
- [[HTTP]] —— 对比文本协议 vs 二进制协议：HTTP 靠 `\r\n` 与 `Content-Length`，SSH 靠 `uint32` 长度 + padding + MAC；两者都是"一连接多路复用"的另一种取向（HTTP/2 stream vs SSH channel）
- [[TLS]] —— 同为"握手协商 + 记录层 + 会话密钥"，可对照 KEX/证书信任 vs DHE/ECDHE/CA 链，以及 ETM vs TLS 1.3 AEAD-only 的演进结论
- [[mqtt]] —— MQTT over TCP + TLS 与 "SSH 隧道转发 1883 端口" 的取舍：前者有协议内建 TLS，后者零改造但引入跳板运维成本

---

## 📚 参考资料

1. **T. Ylonen, C. Lonvick (Eds.)** — *The Secure Shell (SSH) Protocol Architecture*, **RFC 4251**, 2006.（三层模型、数据类型 uint32/boolean/string/name-list/mpint）
2. **T. Ylonen, C. Lonvick (Eds.)** — *The Secure Shell (SSH) Authentication Protocol*, **RFC 4252**, 2006.
3. **T. Ylonen, C. Lonvick (Eds.)** — *The Secure Shell (SSH) Transport Layer Protocol*, **RFC 4253**, 2006.（§6 二进制报文、§7 KEX 密钥派生、350 KB 包长上限）
4. **T. Ylonen, C. Lonvick (Eds.)** — *The Secure Shell (SSH) Connection Protocol*, **RFC 4254**, 2006.（channel 类型、窗口流控、`session`/`direct-tcpip`/`forwarded-tcpip`）
5. **M. Johnston, S. Naccache** — *The Secure Shell (SSH) Public Key File Format*, **RFC 4716**, 2006.
6. **P. Austin** — *SSH User Authentication by Keyboard-Interactive Method Using a Single Challenge*, **RFC 4256**, 2006.
7. **C. Lonvick (Eds.)** — *SSH Message Numbers and Name Space*, **RFC 4250**, 2006.
8. **T. Ylonen** — *The Secure Shell (SSH) Transport Layer Encryption Modes*, **RFC 4344**, 2006.（CTR 模式、报文截断攻击分析、序号不回卷）
9. **H. Harkins, D. Carrel** — *The Oakley Key Determination Protocol*, **RFC 2409** / *Negotiated MODP Groups* **RFC 3526**, 2003.（DH GEX 所用的安全素数群）
10. **S. Josefsson** — *Diffie-Hellman Group Exchange for the Secure Shell (SSH) Transport Layer Protocol*, **RFC 4419**, 2006.
11. **A. Green, P. Johns** — *Elliptic Curve Algorithm Integration in the Secure Shell*, **RFC 5656**, 2010；**J. IETF SSHM / Saltzman**, *curve25519-sha256 Key Exchange*, **RFC 8709**, 2020.
12. **A. Kuehn** — *Extension Negotiation in the Secure Shell (SSH) Protocol*, **RFC 8308**, 2018.
13. **J. Yoder** — *HMAC in SSH* / **RFC 6668**, *A Sign and Seal Algorithm for the Secure Shell*, 2012.
14. **OpenBSD Project** — *OpenSSH 源码与文档*：`src/usr.bin/ssh/PROTOCOL`、`PROTOCOL.key`、`PROTOCOL.agent`、`packet.c`、`kexdh.c`、`auth2-pubkey.c`；手册页 `ssh(1)`、`sshd(8)`、`sshd_config(5)`、`ssh_config(5)`、`ssh-keygen(1)`、`sshd -Q` 能力查询.
15. **S. Barber, C. Lonvick** 等 — *SSH Security FAQ / NIST SP 800-123 Guide to General Server Security*（服务器与密钥管理加固建议）.
