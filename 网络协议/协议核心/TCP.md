---
title: TCP 协议完整技术手册
tags:
  - Network
  - Protocol
  - TCP
  - Transport
  - Congestion-Control
  - Socket
status: Completed
---

# 🔗 TCP 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：TCP（Transmission Control Protocol，传输控制协议）运行在传输层，提供**面向连接（connection-oriented）、端到端、可靠（reliable）、有序、无重复、基于字节流（byte-stream）**的双向传输服务；可靠性由**序号 + 累积确认 + 超时/快速重传 + 滑动窗口（流量控制与拥塞控制）**四套机制共同保证。
> - **内容范围**：协议定位与分层 · 首部逐字段 · 三次握手与四次挥手 · 11 状态机 · 可靠传输与 RTO · 流量控制（rwnd/ZWP/Nagle）· 拥塞控制（cwnd 四阶段与数值演化）· 关键参数与排错清单
> - **推荐路径**：§1 定位 → §2 首部 → §3 握手/挥手 → §4 状态机 → §5 可靠传输 → §6 流量控制 → §7 拥塞控制 → §8 参数与排错 → §9 图

---

## 🧭 1. 协议定位与分层关系

### 1.1 核心属性

| 属性 | 说明 |
| :--- | :--- |
| 定义 | TCP（Transmission Control Protocol），规范主线：RFC 793（旧）→ **RFC 9293**（现行） |
| 所在层 | 传输层 / Transport Layer（OSI 第 4 层，TCP/IP 模型传输层） |
| 连接性 | **面向连接**：通信前必须三次握手（Three-Way Handshake）建立连接，结束后四次挥手释放 |
| 数据单位 | **字节流（Byte Stream）**，无消息边界；应用侧的 `write()` 与对端 `read()` 数量互不相关 |
| 可靠性 | 确认重传 + 序号排序 + 校验和 + 滑动窗口，交付语义「**不丢、不重、不乱序**」 |
| 双工 | **全双工（Full-Duplex）**：一条连接两个方向各自维护独立的 Seq/Ack 与窗口状态 |
| 连接标识 | 四元组（源 IP, 源端口, 目的 IP, 目的端口）+ 协议号 6；加上接收方序号空间构成 socket |
| 承载上层 | HTTP/1.1、HTTP/2、TLS、SSH、FTP、SMTP、MySQL、Redis、MQTT（over TCP）等 |
| 状态位置 | **只存在于两端主机**，中间路由器无连接状态（区别于 MPLS/X.25 虚电路） |

### 1.2 与 UDP / IP 的分层关系

```text
┌───────────────────────────────────────────────────────────┐
│  应用层  HTTP / TLS / SSH / FTP / MQTT                    │
├───────────────────────────────────────────────────────────┤
│  传输层  ┌────────────── TCP ──────────────┐ ┌── UDP ──┐  │  ← 端到端、进程到进程（端口）
│          │ 连接管理 / 可靠传输 / 流控 / 拥塞 │ │  无连接  │  │
│          └──────────────────────────────────┘ └─────────┘  │
├───────────────────────────────────────────────────────────┤
│  网络层  IP（尽力而为 / 无序 / 可重复 / 可丢失 / 分片）    │  ← 逐跳、主机到主机
├───────────────────────────────────────────────────────────┤
│  链路层  以太网 / Wi-Fi（CRC、MTU、ARP）                   │
└───────────────────────────────────────────────────────────┘
```

> [!NOTE] TCP 的「可靠性」建立在 IP 的「不可靠」之上
> IP 层只提供 best-effort 服务：可丢包、可乱序、可重复、MTU 有限。TCP 在端系统上用**序号（Seq）、确认号（Ack）、重传定时器、校验和**在不可靠信道之上「构造」出可靠字节流；**时延与带宽上界仍然无法保证** —— 网络彻底中断时 TCP 只能退避重传直到超时断开。

### 1.3 TCP 与 UDP 对照

| 维度 | TCP | UDP |
| :--- | :--- | :--- |
| 连接 | 面向连接（握手 1 RTT） | 无连接，直接发 |
| 语义 | 字节流 | 数据报（保留消息边界） |
| 可靠性 | 确认 + 重传 + 排序 | 不保证（尽力而为） |
| 有序性 | 严格有序交付 | 到达顺序 = 网络顺序 |
| 拥塞控制 | 有（cwnd） | 无（需应用自建，如 QUIC/RUDP） |
| 首部开销 | 20–60 字节 | 8 字节 |
| 队头阻塞 | 有（丢 1 个包后面全部等待重传） | 无 |
| 首包时延 | 1 RTT（+TLS 另计） | 0 RTT |
| 典型场景 | 网页、文件、数据库、RPC | DNS、实时音视频、QUIC、游戏 |

详见 [[UDP]]。

---

## 📦 2. TCP 首部逐字段解析（Header Format）

### 2.1 总体布局

首部**固定 20 字节（5 行 × 32 bit）**，加**选项（Options）+ 填充（Padding）**后最长 60 字节；`Data Offset` 以 4 字节为单位表示首部总长。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|        Source Port (16)       |    Destination Port (16)     |   行 1
+-------------------------------+-------------------------------+
|                     Sequence Number (32)                     |   行 2
+--------------------------------------------------------------+
|                   Acknowledgment Number (32)                 |   行 3
+--------------+----------------+---------------+--------------+
| Data Offset  | Reserved (6/4) | Flags (6/8)   |   Window(16)  |   行 4
+--------------+----------------+---------------+--------------+
|          Checksum (16)        |      Urgent Pointer (16)     |   行 5
+-------------------------------+------------------------------+
|            Options (0–40 字节，变长，需补齐 32 bit 边界)      |   行 6+
+--------------------------------------------------------------+
|                    Data（上层字节流载荷）                     |
+--------------------------------------------------------------+
```

### 2.2 字段逐条说明

| 字段   |               位宽                | 名称（英文）                         | 作用                                                   | 排错/工程要点                                                                           |
| :--- | :-----------------------------: | :----------------------------- | :--------------------------------------------------- | :-------------------------------------------------------------------------------- |
| 源端口  |               16                | Source Port                    | 发送方 socket 端口，1–65535（0 保留）                          | 临时端口范围 `net.ipv4.ip_local_port_range`（默认 32768–60999）；客户端端口耗尽常因并发过高               |
| 目的端口 |               16                | Destination Port               | 接收方服务端口                                              | `ss -ltnp` 核对监听端口与进程；防火墙/NAT 改写会导致连接被重置                                           |
| 序号   |               32                | Sequence Number, Seq           | 本段数据**第一个字节**在发送方字节流中的编号；SYN/FIN 各消耗 1 个序号（不携带数据也占位） | 32 bit 回绕（wrap around）：1 Gbps 约 17 分钟绕一圈；靠 **PAWS**（时间戳选项）识别陈旧重复段                 |
| 确认号  |               32                | Acknowledgment Number, Ack     | 期望收到对方下一字节的序号 = 已成功累积收到的最高连续序号 + 1                   | 仅当 `ACK` 标志置位才有效；**累积确认（cumulative ACK）**导致乱序场景产生重复 ACK（DupACK）                   |
| 数据偏移 |                4                | Data Offset / Header Length    | 首部长度，单位 **4 字节**；纯首部 = 5（20 字节），最大 15（60 字节）         | 抓包里显示 `data offset: 8` 说明带 12 字节选项（典型：MSS 4 + SACK-perm 4 + TS 10→补齐）             |
| 保留   |     6（RFC 793）/ 4（RFC 9293）     | Reserved                       | 置 0；RFC 854 曾提议 Nonce 位，未普及                          | 中间盒（middlebox）若改写此区导致 RST，是云环境疑难杂症来源之一                                            |
| 标志位  | 6（RFC 793）+ ECN 2 = 8（RFC 9293） | Flags / Control Bits           | 见 §2.3 逐位解释                                          | `tcpdump` 输出中 `S`=SYN、`F`=FIN、`.`=无、`P`=PSH、`R`=RST，如 `S.`、`F.`、`.`               |
| 窗口   |               16                | Window Size, Advertised Window | 接收方还能接收多少字节（**rwnd**），从 Ack 处开始计算                    | 只有 16 bit → 上限 65535 字节；必须靠**窗口缩放（Window Scaling）**选项突破「带宽时延积」限制                  |
| 校验和  |               16                | Checksum                       | 覆盖**伪首部 + TCP 首部 + 数据**的 16 bit 反码和                  | 能查多数误码，**不具备密码学强度**；现代依赖网卡卸载（TSO/GRO/L4 checksum offload），抓包看到的 `incorrect` 常为假告警 |
| 紧急指针 |               16                | Urgent Pointer                 | 指向紧急数据末端的偏移，配合 `URG` 位                               | **实践中已废弃**（BSD/主流栈未实现真正 out-of-band 语义），`TCP_MAXSEG`/`send(MSG_OOB)` 不要依赖         |
| 选项   |        0–320（补齐 32 bit）         | Options                        | MSS、窗口缩放、SACK、时间戳等                                   | 仅在 **SYN 报文**中协商 MSS / WS / SACK-permitted；`nop`(kind 0) 用于对齐，`eol`(kind 1) 结束    |

### 2.3 标志位（Flags）逐位解释

6 个基本控制位按位从高到低写作助记 **U‑G‑R‑A‑B‑F** 一类口诀，展开即 **URG / ACK / PSH / RST / SYN / FIN**（RFC 9293 在此之上向前扩展 ECE、CWR 两位，构成 8 bit）：

| 位序 | 标志 | 助记字母 | 全称 | 含义 | 典型出现 |
| :--: | :--- | :--: | :--- | :--- | :--- |
| 2 | **CWR** | —（ECN） | Congestion Window Reduced | 我（发送方）已经因为收到 ECE 而缩小了拥塞窗口 | ECN 拥塞通知（RFC 3168） |
| 3 | **ECE** | —（ECN） | ECN-Echo | 对端 IP 首部被标记 CE（拥塞遭遇），请降速 | ECN（`tcp_ecn=2` 时仅对自建连接启用） |
| 4 | **URG** | **U** | Urgent | 紧急指针字段有效 | 已废弃，几乎恒为 0 |
| 5 | **ACK** | **A** | Acknowledgment | 确认号有效 | 除首个 SYN 外全部置 1；`ctrl=0x10` |
| 6 | **PSH** | **B**（Push） | Push | 提示接收方立即把数据交付应用，别攒在缓冲里 | 交互式协议常见（`send()` 尾部），实现基本忽略其语义 |
| 7 | **RST** | **R** | Reset | 异常/强制复位连接，丢弃待发数据 | 端口未监听、半开连接被对端拒绝、防火墙干预 |
| 8 | **SYN** | **S** | Synchronize | 同步初始序号 ISN，请求建立连接 | 握手第 1、2 包（`ctrl=0x02` / `0x12`） |
| 9 | **FIN** | **F** | Finish | 本方向字节流发送完毕，请求释放 | 挥手（`ctrl=0x01`，常与 ACK 合为 `0x11`） |

> [!TIP] ctrl 数值速记
> `tcpdump -v` 里的 `flags [tcp]` 十六进制：`0x02` SYN、`0x12` SYN+ACK、`0x10` ACK、`0x18` PSH+ACK、`0x11` FIN+ACK、`0x04` RST、`0x14` RST+ACK。

### 2.4 选项（Options）

| Kind | 选项 | 长度 | 出现时机 | 作用 |
| :--: | :--- | :--: | :--- | :--- |
| 0 / 1 | `EOL` / `NOP` | 1 | 任意 | 选项表结束 / 4 字节对齐填充 |
| 2 | **MSS**（Maximum Segment Size） | 4 | 仅 SYN | 本端能接收的最大 TCP 载荷；默认宣告 1460（MTU 1500） |
| 3 | **Window Scale**（WS，RFC 7323/1122） | 3 | 仅 SYN | 窗口缩放因子 $s$（0–14），实际 rwnd = 首部 Window $\ll s$；16 bit 上限扩到约 1 GB |
| 4 | **SACK-Permitted** | 2 | SYN | 支持选择确认 |
| 5 | **SACK**（RFC 2018） | 8–40 | 数据包 | 报告「已收到的非连续块」边界，一次重传补齐多空洞 |
| 8 | **Timestamps**（TSval / TSecr，RFC 7323） | 10 | 除 SYN 外可含 | RTT 精确测量 + PAWS 防回绕序号误判 |
| 28 | UB（快速打开 / 实验）等 | 变长 | — | TCP Fast Open 用 `TFO cache hash` 携带 cookie 概念，见 RFC 7413 |

### 2.5 伪首部校验（Pseudo-Header）

TCP 校验和不包含真实链路信息，因此计算时**临时拼一段「伪首部」**，把 IP 层的身份与长度纳入，从而发现**误投递到错误 socket** 的报文：

```text
IPv4 伪首部（12 字节）            IPv6 伪首部（40 字节）
+----------------+---------------+   +----------------+
| 源 IP 地址     | 32 bit        |   | 源 IP          | 128 bit
| 目的 IP 地址   | 32 bit        |   | 目的 IP        | 128 bit
| 零 + 协议(=6)  | 16 bit        |   | TCP 长度       | 32 bit
| TCP 长度       | 16 bit        |   | 零(24)+NextHdr | 32 bit（54/6）
+----------------+---------------+   +----------------+
```

计算规则：把首部 + 数据按 16 bit 分组做**一补数（one's complement）求和**，取反填入；校验结果为 `0xFFFF`（等价于求和溢出后为全 1）。长度为奇数时在末尾虚拟补一个 0 字节（不发送）。

> [!WARNING] 校验和的两大坑
> 1. **网卡卸载**：开启 `tx-checksum-ip-generic`/LRO/GRO 后，抓包时校验和由内核填 0 或标记 `bad`，`Wireshark` 需勾选 *"Validate TCP checksum"* 才不误报。
> 2. **不能防篡改**：需要完整性/机密性时，安全必须上移到 TLS 层，见 [[HTTP]] 与 TLS 相关笔记。

---

## 🤝 3. 三次握手与四次挥手

### 3.1 三次握手（Connection Establishment）

设**客户端初始序号 ISN = $x$ = 1000**，**服务端初始序号 ISN = $y$ = 5000**（ISN 由时钟 + 四元组哈希随机化，防历史连接混淆，RFC 6528）：

| # | 方向 | 标志（ctrl） | Seq | Ack | Window | 选项 | 客户端状态 | 服务端状态 |
| :--: | :--- | :--- | :--: | :--: | :--: | :--- | :--- | :--- |
| ① | C → S | **SYN**（0x02） | 1000 | — | 64240 | MSS=1460, WS=7, SACK-perm, TS | `CLOSED` → `SYN_SENT` | `LISTEN` |
| ② | S → C | **SYN + ACK**（0x12） | 5000 | **1001** | 65535 | MSS=1460, WS=7, SACK-perm, TS | `SYN_SENT` | `LISTEN` → `SYN_RCVD` |
| ③ | C → S | **ACK**（0x10） | **1001** | **5001** | 65535 | （TS） | `SYN_SENT` → **`ESTABLISHED`** | `SYN_RCVD` → **`ESTABLISHED`**（移入全连接队列，被 `accept()` 取走） |

**为什么必须是三次？**

1. **双向序号同步**：每个方向都需要告知并确认自己的 ISN，两次只能让服务端确信客户端可达。
2. **防止历史（陈旧）连接**：被延迟的旧 SYN 若两次即建立，服务端会凭空挂起一条无用的半连接并浪费资源（ARQ 二义性）。
3. **避免双方同时立即发数据**（握手两次可能形成「连接打开死锁」）。

> [!NOTE] SYN 交换的是「序号空间」而非「窗口」
> 序号在 SYN 上就已经开始消耗（SYN 占 1 个 Seq），所以第 ③ 步的 `Seq=1001`；第一个携带真实数据的包 Ack 仍是 5001，FIN 同样消耗 1 个序号。

### 3.2 半连接队列 / 全连接队列与 SYN Flood

| 队列 | 内核参数 | 入队时机 | 观测手段 |
| :--- | :--- | :--- | :--- |
| **SYN queue（半连接）** | `net.ipv4.tcp_max_syn_backlog` | 收到 SYN、回 SYN+ACK 后 | `ss -tan state syn-recv`；`netstat -s \| grep -i "SYNs to LISTEN"` |
| **Accept queue（全连接）** | `min(backlog, net.core.somaxconn)` | 收到第 ③ 个 ACK 后 | `ss -ltn` 的 `Recv-Q`(已排队未被 accept) / `Send-Q`(队列上限) |

**SYN Flood**：攻击者伪造源 IP 只发 SYN 不回 ACK，半连接队列被占满 → 正常请求 `Connection refused / timeout`。缓解手段：

- **SYN Cookies**（`net.ipv4.tcp_syncookies=1`）：半连接队列满时用源/目的端口与 IP 哈希编码进 SYN+ACK 的 ISN，第 ③ 包回来时验证，**无状态**抗洪。
- 增大 `tcp_max_syn_backlog`、减小 `tcp_synack_retries`（默认 5→重传约 63 s）。
- 上游清洗 / `iptables -m limit`、`nf_conntrack_tcp_timeout_syn_recv` 收紧。
- 云 LB 侧终结握手（TLS/TCP offload）。

### 3.3 四次挥手（Connection Termination）

数据已结束（客户端最后一段数据到 `Seq=2460`），由**客户端主动关闭**：

| # | 方向 | 标志（ctrl） | Seq | Ack | 触发状态变化 | 说明 |
| :--: | :--- | :--- | :--: | :--: | :--- | :--- |
| ① | C → S | **FIN + ACK**（0x11） | **2461** | 5001 | C: `ESTABLISHED` → `FIN_WAIT_1` | 主动关闭方（`close()`/`shutdown(SHUT_WR)`） |
| ② | S → C | **ACK**（0x10） | 5001 | **2462** | C: `FIN_WAIT_1` → `FIN_WAIT_2`；S: → `CLOSE_WAIT` | **半关闭（half-open）**：服务端仍可继续发数据 |
| ③ | S → C | **FIN + ACK**（0x11） | **5001** | 2462 | S: `CLOSE_WAIT` → `LAST_ACK` | 服务端应用关闭后发出 |
| ④ | C → S | **ACK**（0x10） | **2462** | **5002** | C: `TIME_WAIT`；S: `LAST_ACK` → `CLOSED` | Ack = 对方 FIN 序号 + 1 |

> [!TIP] 为什么挥手是四次而握手是三次
> 握手中服务端的「ACK 对方序号」与「同步自己的序号」可以合并成一个 SYN+ACK 段；挥手时收到 FIN 后，应用可能**还有数据要发**（半关闭），因此 ACK 与自己真正的 FIN 通常分属两段 —— 若服务端此刻无待发数据，Linux 也常见 **FIN+ACK 与 ACK 合并成三次挥手**的现象（`tcp_fin_coalesce`/自动合并）。

### 3.4 TIME_WAIT 与 2MSL

主动关闭方发出最后一个 ACK 后进入 `TIME_WAIT`，持续 **2 × MSL**（Maximum Segment Lifetime；RFC 9293 建议 MSL = 2 min，**Linux 固定 60 s** 后回收）：

1. **保证最后一个 ACK 可达**：若 ④ 丢失，对端超时重发 ③（FIN），本端必须还在，能重新回 ACK，否则对端永远停在 `LAST_ACK`。
2. **让旧连接的报文在网络中消亡**：防止同一四元组新连接（端口复用）收到上一连接的迟到重复段（与 PAWS 配合）。

**大量 TIME_WAIT 的应对**：

| 手段 | 说明 | 风险 |
| :--- | :--- | :--- |
| 让**客户端**主动关闭 | 把 TIME_WAIT 压到数量多但短命的一方之外（通常是客户端） | 需要协议/框架配合 |
| 连接池 / keep-alive / HTTP 长连接 | 从根本减少连接建立与拆除次数（首选） | 需处理空闲超时 |
| `net.ipv4.tcp_tw_reuse=1` | 客户端出向连接可复用 TIME_WAIT 的 socket（**依赖时间戳**） | 必须同时开 `tcp_timestamps` |
| `net.ipv4.ip_local_port_range` 放宽 | 扩大端口池 | 治标 |
| ~~`tcp_tw_recycle`~~ | 已被内核移除（NAT 环境丢包） | **禁止再使用** |

### 3.5 CLOSE_WAIT 堆积的含义

`CLOSE_WAIT` 只出现在**被动关闭方**：已收到对端 FIN 并回了 ACK，但**本端应用还没调用 `close()`**。因此：

> [!CAUTION] 看到 CLOSE_WAIT 持续增长 = 本端代码 bug
> 典型原因：① 异常分支未关闭 socket / 文件描述符泄漏；② 业务线程阻塞（连接池归还遗漏、死锁、下游超时未设置）；③ 只处理了「读返回 0」却没关连接。
> 定位：`ss -tanp state close-wait` 找到持有者进程 → `lsof -p <pid> \| wc -l` 核对 fd 数 → 检查应用侧 finally/defer 关闭路径。它**不会**靠内核自动消退，最终撑爆 fd 上限（`Too many open files`）。

---

## 🚦 4. TCP 状态机（11 状态）

### 4.1 状态清单

| 状态 | 归属 | 含义 | 常见停留时长/异常信号 |
| :--- | :--- | :--- | :--- |
| `CLOSED` | 双方 | 初始/终止态，TCB（控制块）不存在 | — |
| `LISTEN` | 服务端 | 等待 SYN，socket 已 `bind` + `listen` | 常驻；没有它 = 未监听/被防火墙拦截 |
| `SYN_SENT` | 客户端 | 已发 SYN，等 SYN+ACK | 短暂；长期堆积 = 对端不可达/被丢弃（SYN 重传最多约 63 s～2 min） |
| `SYN_RCVD` | 服务端 | 已回 SYN+ACK，等最后一个 ACK | 堆积 = SYN Flood 或客户端不可达 |
| `ESTABLISHED` | 双方 | 连接就绪，可全双工收发 | 正常业务态 |
| `FIN_WAIT_1` | 主动方 | 已发 FIN，等 ACK 或 FIN | 短暂 |
| `FIN_WAIT_2` | 主动方 | 已收到 ACK，等对端 FIN（半关闭） | 堆积 = 对端不发 FIN；`tcp_fin_timeout`（默认 60 s）回收 |
| `FIN_WAIT_2` 的孤儿 | — | 应用已 `close()` 但连接仍在内核 | 计入 `tcp_orphan` 限额 |
| `CLOSING` | 主动方 | **同时关闭**：FIN 之后先收到对端 FIN（未带 ACK） | 罕见 |
| `TIME_WAIT` | 主动方 | 等 2MSL 后静默关闭 | 高并发短连接服务的常态（万级） |
| `LAST_ACK` | 被动方 | 已发 FIN，等最后 ACK | 短暂；堆积 = 对端 ACK 丢失或已消失 |
| `CLOSE_WAIT` | 被动方 | 收到 FIN 并 ACK，等本端应用 `close()` | **堆积 = 应用未关连接（bug）** |

（表内「FIN_WAIT_2 的孤儿」是同一状态的资源属性，标准 11 态为：`CLOSED`、`LISTEN`、`SYN_SENT`、`SYN_RCVD`、`ESTABLISHED`、`FIN_WAIT_1`、`FIN_WAIT_2`、`CLOSING`、`TIME_WAIT`、`LAST_ACK`、`CLOSE_WAIT`。）

### 4.2 状态转换表

| 当前状态 | 事件 / 收到 | 动作 | 下一状态 | 典型发起方 |
| :--- | :--- | :--- | :--- | :---: |
| `CLOSED` | 应用 `connect()` | 发 SYN | `SYN_SENT` | 客户端 |
| `CLOSED` | 应用 `bind()`+`listen()` | 建立监听 TCB | `LISTEN` | 服务端 |
| `LISTEN` | 收到 SYN | 发 SYN+ACK，入半连接队列 | `SYN_RCVD` | 服务端 |
| `SYN_SENT` | 收到 SYN+ACK | 发 ACK | `ESTABLISHED` | 客户端 |
| `SYN_SENT` | 收到 RST / 超时耗尽 | 释放 TCB | `CLOSED` | 客户端 |
| `SYN_RCVD` | 收到合法 ACK | 移入全连接队列 | `ESTABLISHED` | 服务端 |
| `ESTABLISHED` | 应用 `close()` | 发 FIN | `FIN_WAIT_1` | 主动方 |
| `ESTABLISHED` | 收到 FIN | 回 ACK，通知应用（read 返回 0） | `CLOSE_WAIT` | 被动方 |
| `FIN_WAIT_1` | 收到 ACK | — | `FIN_WAIT_2` | 主动方 |
| `FIN_WAIT_1` | 收到 FIN（同时关闭） | 回 ACK | `CLOSING` | 主动方 |
| `FIN_WAIT_1` | 收到 FIN+ACK | 回 ACK | `TIME_WAIT` | 主动方（合并路径） |
| `FIN_WAIT_2` | 收到 FIN | 回 ACK | `TIME_WAIT` | 主动方 |
| `CLOSING` | 收到 ACK | — | `TIME_WAIT` | 主动方 |
| `TIME_WAIT` | 2MSL 计时到 | 删除 TCB | `CLOSED` | 主动方 |
| `CLOSE_WAIT` | 应用 `close()` | 发 FIN | `LAST_ACK` | 被动方 |
| `LAST_ACK` | 收到 ACK | — | `CLOSED` | 被动方 |
| 任意 | 收到 RST / 应用异常 | 立即销毁 TCB | `CLOSED` | 双方 |

```text
客户端（主动关闭）                     服务端（被动关闭）
CLOSED ─connect()→ SYN_SENT ──┐        CLOSED →(bind/listen)→ LISTEN
                              │ SYN+ACK ↑│ SYN
                              ↓          ↓ ACK
                         ESTABLISHED ←→ ESTABLISHED
   send/recv 全双工数据 ────────────────────────────
       ↓ close(): FIN          ↑ close(): FIN
FIN_WAIT_1 ──ACK──→ FIN_WAIT_2  ↑      CLOSE_WAIT
       │  收到 FIN（或同时关闭 →CLOSING）│  ↑ 收到对端 FIN 后回 ACK
       ↓                              ↓  ↑ 本端 close()
TIME_WAIT ←──── ACK ─────────── LAST_ACK
   │ 2MSL(60s)                       │ 收到 ACK
   ↓                                 ↓
CLOSED                            CLOSED
```

---

## 🎯 5. 可靠传输机制（Reliable Data Transfer）

### 5.1 序号与确认

- **Seq**：本段首个字节的流编号；**Ack** = 已连续收到的最高序号 + 1（**累积确认**）。
- 每段数据带序号，接收方按序号**排序入队**，遇到空洞则缓存乱序段并持续回**重复 ACK（DupACK）**+ SACK 块。
- 序号空间 32 bit，回绕（wrap-around）时靠**时间戳选项 + PAWS**丢弃迟到段（RFC 7323）。
- **重传段仍占序号但不再新计数**：`Retransmitted segment` 的 Seq 与原段相同。

### 5.2 重传超时 RTO：Jacobson / Karels 算法（RFC 6298）

对每个未被重传的报文采样 RTT $R$，维护平滑值与偏差值：

$$
\begin{aligned}
\mathrm{SRTT} &\leftarrow (1-g)\cdot \mathrm{SRTT} + g\cdot R && g = 1/8\\[2pt]
\mathrm{RTTVAR} &\leftarrow (1-h)\cdot \mathrm{RTTVAR} + h\cdot |\mathrm{SRTT} - R| && h = 1/4\\[2pt]
\mathrm{RTO} &\leftarrow \mathrm{SRTT} + k\cdot \mathrm{RTTVAR} && k = 4
\end{aligned}
$$

| 约束 | 取值 |
| :--- | :--- |
| 首包未采样时初始 RTO | Linux `TCP_TIMEOUT_INIT` = **1 s**（RFC 6298 建议 1 s，旧值 3 s） |
| RTO 下界 | Linux `TCP_RTO_MIN` = **200 ms**（RFC 6298 建议 ≥ 1 s） |
| RTO 上界 | Linux `TCP_RTO_MAX` = **120 s** |
| 重传退避 | 每次重传 $\mathrm{RTO} \leftarrow 2 \times \mathrm{RTO}$（**指数退避**，最多重传 `tcp_retries2`=15 次，约 13–30 min 判死） |

> [!IMPORTANT] Karn 算法（Karn-Partridge）
> 1. **不对重传报文做 RTT 采样** —— 因为无法区分收到的 ACK 对应原包还是重传包（二义性）。
> 2. 只采纳**未发生重传**的报文 RTT 样本。
> 3. 退避：若某段被重传，其 RTO 加倍后再计时。
> 4. 长时间无新样本时 RTO 会「冻结」，需配合 **clocking / 定期探测** 恢复。

### 5.3 快重传与快恢复（RFC 5681）

- **触发**：收到 **3 个重复 ACK**（即第 4 个相同 Ack 号）→ 判定该段丢失，**立即重传，不等 RTO**。
- **快恢复**：`ssthresh = max(cwnd / 2, 2*SMSS)`；Reno 下 `cwnd = ssthresh + 3*SMSS`（每收到一个 DupACK 再加 1 MSS，「拥塞避免的伪装」），收到新 ACK 后 `cwnd = ssthresh`。
- **RTO 超时的区别**：超时说明管道近乎清空 → `ssthresh = cwnd/2`，`cwnd = 1 MSS`（实际用 **IW = 10 MSS**，RFC 6928 / Linux `TCP_INIT_CWND=10`）后**重新慢启动**（这是快恢复与超时恢复的核心差别）。
- **SACK（RFC 2018）**：DupACK 里携带「已收到的不连续块」，发送方可**一次重传所有已知空洞**（而非 Reno 的每次一个），并把 SACK 信息纳入 `cwnd` 的「拥塞窗口避免过冲」校正（RFC 6675 的 `Pipe`/`retransmitted bytes` 记账）。
- **RACK-TLP（RFC 8985）**：以**时间**而非丢包计数判定重传（`loss = (SACKed | latest_sent) - rtort`），并在只有尾部丢包时发送 **TLP（Tail Loss Probe，尾丢失探测包）**：一次正常 RTT 后额外发送 1 个探测段，用一次 RTT 的代价换来快重传，避免退到昂贵的 RTO。

### 5.4 重传类型速查

| 类型 | 触发 | 说明 |
| :--- | :--- | :--- |
| `timeout retransmit` | RTO 到期 | cwnd 跌到 IW，代价最高 |
| `fast retransmit` | 3 DupACK / RACK 判丢 | cwnd 减半 |
| `RTO-retransmit / SYN-retransmit` | 握手中超时 | `ss -ti` 可见 `retrans:` 计数 |
| `TLP` | 尾部数据无 ACK | 探测以触发 ACK |
| `Delayed ACK` | 定时器 40 ms | 见 §6.3 |
| `SACK retransmit` | SACK 块指示空洞 | 精准补洞 |

---

## 🌊 6. 流量控制（Flow Control：rwnd）

### 6.1 接收窗口

流量控制解决「**发送方把接收方缓冲区打爆**」的问题，与拥塞控制（保护网络）正交：

$$
\text{可发送字节（可用发送窗口）} = \min(\underbrace{\text{rwnd}}_{\text{接收方通告}},\ \underbrace{\text{cwnd}}_{\text{拥塞窗口}}) - \underbrace{\text{in-flight}}_{\text{已发未确认}}
$$

| 概念 | 说明 |
| :--- | :--- |
| **rwnd（Received Window）** | 对端 ACK 首部 `Window` 字段（经 WS 缩放后）通告的剩余接收空间 |
| **应用缓冲区** | 内核 `sk_rcvbuf` 与 socket 选项 `SO_RCVBUF`；`tcp_rmem`（min/default/max 自动调优）；慢消费者 → rwnd 收缩 → 端到端背压 |
| **通告时机（有效窗口更新）** | 通常合并进 ACK；Linux 只在窗口增长 ≥ $\min(\text{接收缓冲}/2,\ \text{MSS})$ 或能通告一个完整 MSS 时发送，避免「 silly window syndrome / SWS 」 |
| **发送侧上限** | `tcp_wmem` / `SO_SNDBUF`；`ss -ti` 中 `rcv_ssthresh`、`unacked`、`inflight` |

### 6.2 零窗口与窗口探测（ZWP, Zero Window Probe）

1. 接收方缓冲满 → 通告 `Window = 0` → 发送方停止发送，启动 **persist timer（零窗口定时器）**。
2. 发送方周期性发 **ZWP**：1 字节（或含 1 字节数据的段）探测，**消耗 1 个序号**，迫使对端必须回 ACK。
3. 对端仍无空间 → 回 `Window = 0`（探测按指数退避，上限 `TCP_PROBE_THRESHOLD`）；一旦有空间 → 回带新窗口的 **Window Update**。
4. **Window Update 丢失 → 死锁**：接收方等数据、发送方等窗口更新。兜底：接收方定期主动发送 Window Update，以及发送方持续 ZWP；Linux 上 `ss -ti` 可见 `zero_win_calls`/`z_wnd_probes` 计数（`nla` 统计项）。

> [!WARNING] 应用侧「读得慢」会毁掉整条链路
> 零窗口/小窗口的根因绝大多数不在网络，而在**消费线程阻塞**（同步落库、下游 RPC 超时、GC 停顿）。用 `ss -tin state established 'sk rmem > 0'` 观察 `rcv_space` 与 `zero_win_calls`，配合 `netstat -s \| grep -i window` 计数增长判定。

### 6.3 Nagle 算法与延迟确认（Delayed ACK）的交互

| 机制 | 规则 | 开关 |
| :--- | :--- | :--- |
| **Nagle** | 若存在**已发出未被确认**的小段，则后续小包先在内核缓存，累计到 ≥ MSS 或所有小包被 ACK 时才发送 | 关闭：`setsockopt(TCP_NODELAY)`（默认 Linux 已对交互型连接设 nodelay） |
| **延迟确认** | 收到数据不立即 ACK，最多等 **40 ms**（`TCP_DELACK_MAX`）或等到有数据可捎带（piggyback）| 尽量即时：`TCP_QUICKACK`（一次性，需重设） |
| **FIN 等待** | `TCP_CORK` / `MSG_MORE`：把多次 `write()` 攒成一个段，用于大响应体 | 用完必须解除，否则永久卡住 |

**最坏交互（教科书案例）**：应用做「请求 → 等待 → 应答」的乒乓（RPC 常见）。A 端 Nagle 攒着小包等 ACK，B 端 Delayed ACK 攒着 ACK 等 40 ms → 双方互等，**每轮往返固定多 ~40 ms**，吞吐跌至约 $\text{MSS}/40\text{ms}$。

> [!TIP] 处置顺序
> 交互式/小消息协议（RPC、Redis、数据库、SSH）→ 明确设 **`TCP_NODELAY`**；大块流式传输 → 保持 Nagle 或改用 `TCP_CORK` 写满 MSS。不要用「两边都关」赌性能，先测 `ss -ti` 的 RTT 与应用时延分位。

---

## 🚗 7. 拥塞控制（Congestion Control：cwnd）

### 7.1 四阶段

| 阶段 | 触发条件 | cwnd 变化规则（每收到 1 个 ACK） | 增长形态 |
| :--- | :--- | :--- | :--- |
| **慢启动 Slow Start** | 连接建立、RTO 后 | $\text{cwnd} \leftarrow \text{cwnd} + \text{SMSS}$（约每 RTT **翻倍**） | 指数，直到 `ssthresh` 或丢包 |
| **拥塞避免 Congestion Avoidance** | $\text{cwnd} \ge \text{ssthresh}$ | $\text{cwnd} \leftarrow \text{cwnd} + \text{SMSS}\cdot\frac{\text{SMSS}}{\text{cwnd}}$（约每 RTT **+1 MSS**） | 线性（AIMD 的 AI） |
| **快重传 Fast Retransmit** | 3 个 DupACK | 立即重传缺失段 | — |
| **快恢复 Fast Recovery** | 快重传之后 | Reno: $\text{cwnd} = \text{ssthresh} + 3\cdot\text{SMSS}$，收到新 ACK → $\text{cwnd} = \text{ssthresh}$ | MD（乘性减）后线性回升 |

$$
\text{ssthresh}_{\text{loss}} = \max\!\left(\frac{\text{cwnd}}{2},\ 2\cdot \text{SMSS}\right)
$$

| 关键变量 | 含义 |
| :--- | :--- |
| `ssthresh`（Slow Start Threshold） | 指数增长 → 线性增长的切换门限；丢包时按上式减半 |
| `cwnd`（Congestion Window） | 网络侧允许的在途字节上限 |
| `IW`（Initial Window） | 初始窗口 = **10 × MSS**（RFC 6928；Linux `TCP_INIT_CWND = 10`） |
| 实际发送上限 | $\min(\text{cwnd}, \text{rwnd})$ —— 两者取小 |

> [!NOTE] 一句话 BBR
> **BBR（Bottleneck Bandwidth and RT-round-trip time）** 不再把丢包当拥塞信号，而是持续估计**链路带宽 BtlBW**与**最小 RTT（RTprop）**，把在途数据钉在 $\text{BDP} = \text{BtlBW} \times \text{RTprop}$ 上（ProbeBW / ProbeRTT 周期），在高带宽高时延或有损链路上显著优于丢包型算法（Reno/CUBIC）。

### 7.2 数值演化示例（IW = 10 MSS，SMSS = 1460 B，ssthresh = 100 MSS）

假设接收窗口足够大（rwnd 不构成瓶颈），按「一个 RTT 内该轮发出的报文全部被确认」为一次增长轮次：

| 轮次（RTT） | 阶段 | 本轮起始 cwnd（MSS） | 每 ACK 增量 | 结果 cwnd | 说明 |
| :--: | :--- | :--: | :--: | :--: | :--- |
| 0 | 建连 | — | — | **10** | 初始窗口 IW=10（约 14.6 KB） |
| 1 | 慢启动 | 10 | +1 MSS/ACK（10 次） | 20 | 每 RTT 翻倍 |
| 2 | 慢启动 | 20 | +20 | 40 | — |
| 3 | 慢启动 | 40 | +40 | 80 | — |
| 4 | 慢启动 → 拥塞避免 | 80 | 到 100 即停 | **100** | 达到 `ssthresh=100`，切换线性 |
| 5 | 拥塞避免 | 100 | 每 RTT +1 | 101 | AIMD 的加性增 |
| 6 | 拥塞避免 | 101 | +1 | 102 | — |
| … | 拥塞避免 | 108 | +1 | 109 | — |
| n | 拥塞避免 | **110** | 收到 **3 个 DupACK** | — | 判定 1 段丢失 → 快重传 |
| n | 快恢复 | — | `ssthresh = 110/2 = 55` | **58**（=55+3） | Reno：DupACK 期间继续 +1/个 |
| n+1 | 拥塞避免 | 收到新 ACK | `cwnd = ssthresh` | **55** | 线性回升（不回到 1，这就是快恢复的价值） |
| n+2 | 拥塞避免 | 55 | +1 | 56 | … 循环 |
| （对照）若走 RTO 超时 | 慢启动 | 任意 | — | **10**，`ssthresh = cwnd/2` | 代价：整条 RTT 曲线从 IW 重来 |

```text
cwnd
 ^
 |                         ／\  (RTO 超时 → 跌到 IW=10，重新慢启动)
 |                        /
 |              慢启动(指数)↗ /↗↗ 线性(拥塞避免)
 |            ／            ↗
 |          ／             /  ← 3×DupACK：ssthresh=c/2, 快恢复到 c/2 后线性
 |        ／              ↘__／↗↗↗
 |      ／  ← ssthresh 门限 - - - - - - - - - - -
 |   IW10／
 +------|----|----|----|----|----|----|----|----|----> RTT
       1    2    3    4    5    6    7    8    9
```

### 7.3 主流算法差异（Linux `net.ipv4.tcp_congestion_control`）

| 算法 | 判据 | 丢包响应 | 备注 |
| :--- | :--- | :--- | :--- |
| **Reno** | 3 DupACK | c/2，恢复期 +1/ACK | 多丢包场景恢复慢 |
| **NewReno** | 同上 | 一个 RTT 内可连续重传多段 | 内核默认历史值 |
| **CUBIC**（RFC 8312，Linux 默认） | 三次函数 | $W(t)=C(t-K)^3+W_{\max}$，$K=\sqrt{W_{\max}\beta/C}$ | 高带宽长肥管道友好、与 RTT 无关的公平性折中 |
| **BBR/BBRv2** | 带宽 + RTT | 不依赖丢包 | 需要公平性/整形注意，与其他算法共存时更激进 |
| **DCQCN / HPCC** | ECN / INT | 数据中心 RDMA 场景 | 见拥塞通知与遥测相关文献 |

---

## 🔧 8. 关键参数与排错清单

### 8.1 MSS 协商与路径 MTU

$$
\text{MSS} = \text{MTU} - \text{IP 首部} - \text{TCP 首部（无选项）}
$$

| 链路 | MTU | MSS（IPv4，纯首部 20 B） | 备注 |
| :--- | :--: | :--: | :--- |
| 标准以太网 | 1500 | **1460** | 最常见 |
| 带时间戳选项（12 B） | 1500 | 1448 | 首部长 32 B（`Data Offset=8`） |
| PPPoE | 1492 | 1452 | 家宽/机房常见陷阱 |
| 隧道（IPIP/GRE/VXLAN） | 1400 / 1450 | 1360 / 1410 | overlay 网络必须预留封装开销 |
| IPv6 以太网 | 1500 | 1440（首部 40 B） | IPv6 中间路由器**不分片**，只能靠 PMTUD |
| Jumbo Frame | 9000 | 8960 | 全链路（含交换机）必须一致，否则黑洞 |

> [!CAUTION] PMTUD 黑洞（Path MTU Discovery Blackhole）
> 若中间设备拦截了 ICMP「需要分片（Type 3 Code 4）」，小包（握手、`GET /`）正常、大包（上传、TLS 证书链）卡死超时。处置：放行 ICMP、改用 `ip no route`/PMTUD 探测、或直接 `ip link set mtu 1400` / `TCP_MAXSEG` 收紧 MSS。诊断：`tracepath -n`、`ping -M do -s 1472 <ip>`。

### 8.2 Keepalive（保活探测）

- 默认**关闭使用**（`SO_KEEPALIVE` 需显式开启），Linux 全局参数：

| 参数 | 默认 | 含义 |
| :--- | :--: | :--- |
| `net.ipv4.tcp_keepalive_time` | 7200 s（2 h） | 空闲多久后开始探测 |
| `net.ipv4.tcp_keepalive_intvl` | 75 s | 探测间隔 |
| `net.ipv4.tcp_keepalive_probes` | 9 | 探测次数，无应答则判定连接死亡并 `ETIMEDOUT` |

- 总判死时间 ≈ `time + intvl × probes` = **2 h 11 min 15 s** —— 对 NAT/LB 空闲超时（常 60–900 s）远远不够，**不要指望它防断连**：应用层心跳（MQTT `keepalive`、gRPC keepalive）或 LB 侧空闲计时才是正解。
- `ss -o state established '( keepalive )'` / `ss -ti` 中的 `keepalive:7200,600,75` 查看实际生效值。

### 8.3 netstat / ss 状态排查清单

| 现象 | 观察命令 | 判读与根因方向 |
| :--- | :--- | :--- |
| 连接建立慢/偶发失败 | `ss -tan state syn-sent`；`netstat -s \| grep -i 'SYNs to LISTEN'` | 半连接队列满 / 对端 down / SYN 被丢；查 `tcp_max_syn_backlog`、`syncookies` |
| accept 不及时 | `ss -ltn`（`Recv-Q` 持续 > 0，接近 `Send-Q`） | 应用 accept 速率不足（线程池/阻塞），溢出会按 `tcp_abort_on_overflow` 决定是否 RST |
| 大量 TIME_WAIT | `ss -s`、`ss -tan state time-wait \| wc -l` | 本端频繁主动关闭短连接 → 上连接池/长连接；确认 `tcp_tw_reuse` 与时间戳 |
| 大量 CLOSE_WAIT | `ss -tin state close-wait` | **应用未 close()**：fd 泄漏、线程卡死；查 `lsof -p` |
| 大量 FIN_WAIT_2 / LAST_ACK | `ss -tan state fin-wait-2` | 对端不发 FIN / 中间设备丢弃 FIN；调 `tcp_fin_timeout` |
| 吞吐上不去 | `ss -ti`（`cwnd:`、`rto:`、`rtt:`、`retrans:`、`pacing_rate`） | cwnd 小 = 丢包或 RTT 大；`unacked`≈`rwnd` = 对端窗口受限 |
| 吞吐呈锯齿/重传多 | `nstat -az \| grep -i retrans`、`tcpdump -i eth0 'tcp[tcpflags] & tcp-push != 0'` | 区分网络丢包 vs 接收端零窗口；看 `TcpExt.TCPRetransSegs` 与 `TcpExt.TCPBacklogDrop` |
| 连接被重置 | `tcpdump 'tcp[tcpflags] & tcp-rst != 0'` | 端口未监听 / LB 健康检查摘除 / 防火墙会话老化 / 半开连接（对端重启） |
| 半开连接悬挂（对端掉电） | `ss -to state established` | 未开 `SO_KEEPALIVE` 也无应用心跳，本地永不知道 |
| 缓冲区不足 | `ss -tm`、`sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem` | 内存收缩或自动调优上限过低 |

**常用一行命令**：

```bash
ss -s                                        # 全局连接状态计数总览
ss -tinp state time-wait                     # TIME_WAIT + 持有进程
ss -ti dst 10.0.0.5                          # 单连接 cwnd/rtt/retrans/pacing
nstat -az | egrep 'TcpRetransSegs|TCPOrigDataSent|TCPTimeouts'   # 重传率
ip route get 1.1.1.1                         # 出接口与生效 MTU（MSS 来源）
ethtool -k eth0 | grep -i checksum          # 校验和卸载（排除假 bad checksum）
```

---

## 🖼️ TCP 首部与握手表时序图

![[TCP.excalidraw]]

---

## 🔗 思考与延伸索引

- [[UDP]] — 无连接对照：为什么 QUIC 选择在 UDP 上重建 TCP 的可靠性
- [[IP]] — 下层尽力而为服务、首部与分片、ICMP 与 PMTUD
- [[HTTP]] — TCP 之上最典型的字节流消费者（长连接、队头阻塞）
- [[SSH]] — 交互式小报文场景，Nagle / 延迟确认时延的经典验证场
- [[TLS]] — 与 TCP 的层间耦合（分片、MTU、握手 RTT 叠加）
- [[MQTT]] — 应用层心跳 vs TCP keepalive 的取舍

---

## 📚 参考资料

1. **IETF** — *RFC 9293: Transmission Control Protocol (TCP)*（现行 TCP 规范与状态机文本）.
2. **IETF** — *RFC 793: Transmission Control Protocol*（原始定义，6 位 Reserved + 6 位 Flags 布局）.
3. **IETF** — *RFC 5681: TCP Congestion Control*（慢启动/拥塞避免/快重传/快恢复，3 DupACK 规则）.
4. **IETF** — *RFC 6298: Computing TCP's Retransmission Timer*（Jacobson/Karels、Karn 规则、RTO 上下界）.
5. **IETF** — *RFC 2018 / RFC 4015: TCP Selective Acknowledgment Options*（SACK 与 DupACK 处理）.
6. **IETF** — *RFC 8985: The RACK-TLP Loss Detection Algorithm for TCP*（时间判丢与尾丢失探测）.
7. **IETF** — *RFC 7323: TCP Extensions for High Performance*（窗口缩放、时间戳、PAWS）.
8. **IETF** — *RFC 6928: Increasing TCP's Initial Window*（IW = 10 MSS 的依据）.
9. **IETF** — *RFC 8312: CUBIC: A New TCP-Friendly Line-Size Growth Function*；*RFC 7661: TCP Alternative Hierarchical Fast Restart* 与 BBR 论文.
10. **IETF** — *RFC 4953: Guidance for Internet Operators to Reduce Vulnerability for DoS* / *RFC 9000（QUIC）* — SYN 洪泛与无连接替代思路.
11. **W. R. Stevens** — *TCP/IP Illustrated, Volume 2: The Implementation*（Addison-Wesley；重传定时器、窗口、状态机源码级分析）.
12. **W. R. Stevens, E. Wright** — *TCP/IP Illustrated, Volume 1: The Protocols*（首部与握手/挥手时序）.
13. **Linux Kernel 文档** — *Documentation/networking/ip-sysctl.rst*、`tcp(7)` man page（`tcp_tw_reuse`、`tcp_keepalive_*`、`TCP_NODELAY`）.
14. **M. Mathis, J. Semke** — *"TCP Congestion Control Making RTT-dependent Measurements"* 与 **Nagle RFC 896**（小包拥塞与交互死锁）.
