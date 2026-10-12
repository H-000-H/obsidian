---
title: UDP 协议完整技术手册
tags:
  - Network
  - Protocol
  - UDP
  - Transport-Layer
  - Checksum
  - Datagram
status: Completed
created: 2026-09-21
---

# 🔗 UDP 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：无连接（Connectionless）、面向数据报（Datagram）、**不保证可靠但提供端到端校验**（Optional Reliability with Mandatory-ish Checksum）的传输层协议；支持单播 / 组播 / 广播，协议本身没有"会话"概念。
> - **内容范围**：协议定位 · 8 字节首部逐字段 · 伪首部与校验和 · 端口分发与数据报尺寸 · TCP/UDP 全面对比 · 典型应用与"应用层补可靠性"（ARQ / 分片重组 / FEC）· 常见坑与抓包假象
> - **推荐路径**：§1 定位 → §2 首部与封装开销 → §3 校验和 → §4 端口与 MTU → §5 对比表 → §6 自建可靠性 → §7 排错清单

---

## 🧭 1. 协议定位：无连接的数据报服务

### 1.1 四个关键词

| 属性 | 说明 |
| :--- | :--- |
| 英文名 | **User Datagram Protocol**，用户数据报协议（RFC 768，1980） |
| 所在层 | 传输层（Transport Layer），IPv4 中由**协议号 17** 标识，IPv6 中由**下一个首部（Next Header）= 17** 标识 |
| 无连接 | 发送前**不握手**、不建会话、不拆连接；`sendto()` 的第一个字节就是应用数据，因此**首个包的时延最低**（0 RTT） |
| 数据报 | **保留消息边界**：一次 `sendto()` = 一个 IP 数据报 = 一次 `recvfrom()` 返回；不像 TCP 那样是字节流（无 sticky/split packet 问题） |
| 不可靠 | 只尽最大努力交付（best-effort）：**可能丢、可能重复、可能乱序、可能延迟**；协议层**无确认、无重传、无序号、无流控、无拥塞控制** |
| 有校验 | 16 bit **反码校验和**覆盖伪首部 + UDP 首部 + 数据，能检出多数 bit 错误；IPv4 下可选（填 0 表示不校验），**IPv6 下强制** |
| 交付形态 | 单播（Unicast）、组播（Multicast，D 类地址 + IGMP）、广播（Broadcast，`255.255.255.255` 受限广播 / 子网定向广播）——**组播/广播是 UDP 独有，TCP 不具备** |
| 拥塞行为 | 内核不做任何退避：**发多快完全由应用与 socket 缓冲决定**，容易"饿死"同链路上的 TCP，也最容易在路由器队列溢出时被丢弃 |

### 1.2 在协议栈中的位置

```text
应用层        │ DNS · DHCP · NTP · RTP/SRTP · QUIC · Syslog · SNMP · 游戏网络层
──────────────┼────────────────────────────────────────────────────────────
传输层 (17)   │ UDP          ← 8 字节首部；进程/套接字可见（socket API）
──────────────┼────────────────────────────────────────────────────────────
网络层        │ IPv4 (20 B)  /  IPv6 (40 B)   ← 分片与重组在这里；尽力而为
──────────────┼────────────────────────────────────────────────────────────
链路层        │ 以太网 (14 B 头 + 4 B FCS) / Wi-Fi / 点对点
──────────────┼────────────────────────────────────────────────────────────
物理层        │ 前导码 + SFD = 8 B（不计入 MTU）
```

### 1.3 套接字 API 语义对照

| 语义 | UDP 表现 |
| :--- | :--- |
| 建连接 | 无（`connect()` 可选，仅在内核里记录对端地址做**入包过滤 + 默认目的地址**，不产生任何报文） |
| 发送 | `sendto()` / `sendmsg()`（无连接）或 `send()` / `write()`（已 `connect()`） |
| 接收 | `recvfrom()` / `recvmsg()`（能拿到源地址）；不匹配 socket 四元组的数据报被内核**静默丢弃**并计数（`UdpNoPorts` / `InCsumErrors`…） |
| 背压 | **没有**：内核收包队列（`sk_rcvbuf`）满即丢弃，不通知发送方；只能靠应用层限速或 `SO_RCVBUF` 扩容 |
| 关闭 | 无 FIN/挥手，`close()` 即释放；对端不会感知（表现为"心跳超时"） |

---

## 📦 2. 首部格式：8 字节定长

### 2.1 首部图（4 × 16 bit）

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          源端口 Source Port        |        目的端口 Destination Port        |  32 bit
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|            长度 Length           |         校验和 Checksum（16 bit）        |  64 bit
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             应用数据 Data（0 ~ 65527 B）                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 2.2 逐字段表

| 字段 | 偏移 | 长度 | 含义 | 关键约定与坑 |
| :--- | :--: | :--: | :--- | :--- |
| **源端口**<br>Source Port | 0 | 16 bit | 发送方端口。 | 可省略（不接收回包时填 0，如部分 DHCP/广播场景）；回包/错误报文会用它作为目的端口。 |
| **目的端口**<br>Destination Port | 2 | 16 bit | 接收方端口。 | 目的主机用它 + 本地 IP 做 socket 分发；无匹配 socket 时**丢包并回 ICMP Port Unreachable**。 |
| **长度**<br>Length | 4 | 16 bit | **UDP 首部 + 数据**的总字节数（**不含** IP 首部）。 | 最小 $8$（只有首部，允许 0 字节数据报）；最大 $65535$。IPv4 下实际受 IP 总长限制：$65535 - 20 = 65515$，即净载最大 **65507 B**。 |
| **校验和**<br>Checksum | 6 | 16 bit | 覆盖**伪首部 + UDP 首部 + 数据**的反码和。 | IPv4 可选，**填 0 = 不校验**；IPv6 **强制非 0**（隧道/封装场景例外）。计算结果恰为 0 时按 RFC 768 **发送全 1（0xFFFF）**（反码中 $-0$ 与 $0$ 等价）。 |

> [!NOTE] 为什么"没有"更多字段
> UDP 首部是**定长 8 字节、无 Options、无 IHL/无序列号/无标志位**——这正是它低开销、低时延、可被硬件快速处理的原因；所有"额外能力"（序号、分片号、会话 ID、时间戳）必须由**应用层自己的首部**承载，见 §6。

### 2.3 封装开销与字节数计算

$$
\underbrace{L_{\text{以太网帧}}}_{\text{线上帧}} = \underbrace{14}_{\text{Eth 头}} + \underbrace{20}_{\text{IPv4 头}} + \underbrace{8}_{\text{UDP 头}} + L_{\text{应用数据}} + \underbrace{4}_{\text{Eth FCS}}
$$

$$
\text{固定协议开销} = 14 + 20 + 8 = \mathbf{42\ B} \quad(\text{以太网} + \text{IPv4} + \text{UDP 首部})
$$

**示例：应用数据 $L = 1000\ \text{B}$（以太网 + IPv4 + UDP 承载）**

| 层 | 该层 PDU 大小 | 本层新增字节 | 累计 |
| :--- | :--- | :--- | :--- |
| 应用 | $1000\ \text{B}$ | 1000 | 1000 |
| UDP 数据报 | $8 + 1000 = 1008\ \text{B}$ | 8（首部） | 1008 |
| IPv4 包 | $20 + 1008 = 1028\ \text{B}$ | 20（首部，无选项） | 1028 |
| 以太网帧 | $14 + 1028 + 4 = 1046\ \text{B}$ | 14（头）+ 4（FCS） | 1046 |
| 含前导码 | $1046 + 8 = 1054\ \text{B}$ | 8（Preamble + SFD） | 1054 |

$$
\eta = \frac{1000}{1046} \approx 95.6\% \qquad
\eta_{\max} = \frac{1472}{14 + 1480 + 4} = \frac{1472}{1498} \approx 98.3\%\ (\text{恰好填满 MTU 1500})
$$

> [!TIP] 快速估算口诀
> - 链路/网络/传输三层头共 **42 B**（IPv4）/ **62 B**（IPv6：$14 + 40 + 8$）。
> - 有效载荷占比 > 95% 的前提是单包 $\gtrsim 800\ \text{B}$；发 60 B 小包的 UDP 业务（如某些遥测/心跳）线上开销占比可达 40% 以上，**要合并（batching）而不是加频率**。

---

## 🛡️ 3. 校验和（Checksum）与伪首部

### 3.1 伪首部（Pseudo-Header）

校验和**不是只算 UDP 首部 + 数据**：发送前会临时拼一段 12 字节的**伪首部**（只参与计算，**不上线、不传输**），用来把"网络层交付信息"绑进校验：

| 字段 | 长度 | IPv4 取值 | IPv6 取值 |
| :--- | :--: | :--- | :--- |
| 源 IP 地址 | 32 bit（4 B） | 源 IPv4 | 源 IPv6（16 B） |
| 目的 IP 地址 | 32 bit（4 B） | 目的 IPv4 | 目的 IPv6（16 B） |
| 全零保留 | 8 bit（1 B） | `0x00` | — |
| 上层协议号 | 8 bit（1 B） | **17**（UDP） | **17**（放在"下一个首部"位置） |
| UDP 长度 | 16 bit（2 B） | UDP 首部 + 数据 | UDP 首部 + 数据 |
| 合计 | **12 B**（IPv4） | | **40 B**（IPv6：$16+16+4$，含 32 bit 长度） |

$$
\text{Checksum} = \overline{\sum\nolimits_{16\text{bit}} \left(\text{伪首部} \,\|\, \text{UDP 首部（校验和字段置 0）} \,\|\, \text{数据（奇数字节补 0）}\right)}
$$

即：按 16 bit 字做**一补码（反码）求和**——每加一次把进位回灌到最低位（end-around carry），最后对 32 bit 中间值折叠（fold）成 16 bit 再**取反**得到校验和。

**接收端验证**：把校验和字段一起纳入求和，结果为全 1（`0xFFFF`）即通过（等价说法：取反后为 0）。

### 3.2 IPv4 可选 vs IPv6 强制

| 场景 | 规则 | 工程后果 |
| :--- | :--- | :--- |
| **IPv4** | RFC 768 允许发送端不算校验和，**填 0 表示"未计算"**。RFC 1122 要求主机**必须实现**该功能，但允许在"上层自带校验"等特殊场景关掉（且必须提供可写开关）。 | 关掉后 UDP 上再无任何差错检测；Linux 上 `setsockopt(IPV4_CHECKSUM...)` 类路径很少用，但 `net.ipv4.udp_checksum` 之类的实现细节/隧道封装会出现 0 校验和包。 |
| **IPv6** | RFC 8200 **强制**：校验和不得为 0（0 视为非法，接收端**丢弃**）。仅两条例外：隧道/封装等"下一跳会重算校验"的场景可显式允许。 | 发 IPv6 UDP 时若计算结果为 0，内核会**改写为 `0xFFFF`**；抓包看到 IPv6/UDP 校验和为 0 一定是异常，Linux 计数 `UdpInCsumErrors`/`Udp6InCsumErrors`。 |
| 组播/链路层已可靠 | 有些实现想省 CPU 而关校验和；以太网 FCS 只保证**单链路**（每跳重算），端到端仍需 UDP 校验和。 | 禁止以"链路可靠"为由关校验和。 |

### 3.3 分片对校验和的影响（易错点）

- 校验和在**发送端分片之前**、对**完整数据报 + 伪首部**一次算好；IP 分片只是把 UDP 数据报按 8 字节边界切开，**不会重算 UDP 校验和**。
- **中间路由器无法校验非首片的 UDP 校验和**（只有首片含 UDP 首部）；IPv4 的分片首部校验和（header checksum）是**逐分片**独立计算的。
- 校验和的**验证只能由目的主机在重组完成后进行**：重组后按伪首部 + 完整数据重算。因此"分片包丢一片"或"重组窗口内混入别的分片"这类问题**校验和检不出来**——IP 标识（Identification）回绕 + 重组超时曾导致过**分片串包投递（IP fragment reassembly misdelivery）**问题。
- 结论：**大 UDP 包 + 分片 = 差错检测能力与交付概率双下降**，优先做应用层分片（每片自带完整 UDP 首部与校验和），见 §6.3。

> [!WARNING] 校验和只能"检错"，不能"纠错"，也检不出这几类错误
> 乱序、重复、延迟投递、把整包放错位置（伪首部之外的逻辑错）——UDP 校验和一律无感。它也不是密码学意义上的完整性：可被伪造者同步重算。需要完整性请用应用层 CRC32C/xxhash（抗随机错误）或 HMAC/AEAD（抗篡改，见 TLS 记录层、QUIC 自带加密）。

---

## 🚪 4. 端口、分发与数据报尺寸

### 4.1 端口与 socket 分发（demultiplexing）

UDP 收包时按**五元组**中的四元组匹配 socket：

$$
(\text{协议}=17,\ \text{本地 IP},\ \text{本地端口},\ \text{远端 IP},\ \text{远端端口})
$$

| socket 状态 | 匹配规则 |
| :--- | :--- |
| 未 `connect()` | 只匹配（本地 IP，本地端口）；本地 IP 为 `0.0.0.0` 时接受任意地址、任意源（wildcard） |
| 已 `connect()` | **额外**要求源 IP/端口与对端一致，不一致的包被丢弃；同时 `send()` 有默认目的地址、能异步收到 ICMP 差错 |
| 匹配失败 | 丢弃数据报 + 计数（`UdpNoPorts`、`UdpRcvbufErrors`）+ 回 **ICMP Port Unreachable**（受限速：`net.ipv4.icmp_ratelimit`、`icmp_errors_use_inbound_ifaddr`） |

**端口号管理（RFC 6335）**：

| 区间 | 范围 | 名称 | 说明 |
| :--- | :--- | :--- | :--- |
| 0 – 1023 | **System Ports**（旧称 Well-known） | 系统端口 | 需 root 或被 `net.ipv4.ip_unprivileged_port_start` 放行；`CAP_NET_BIND_SERVICE` |
| 1024 – 49151 | **User Ports**（Registered） | 注册端口 | IANA 登记，业务端口常用（如 53/67/68/123/514/161） |
| 49152 – 65535 | **Dynamic/Private Ports** (Ephemeral) | 动态端口 | 客户端临时端口；受 `net.ipv4.ip_local_port_range` 控制（Linux 默认 32768–60999） |

> [!NOTE] 端口只有 16 bit
> $2^{16} - 1 = 65535$ 个可分配端口（0 保留），这是"单 IP 上服务/会话数量上限"的根因之一：客户端并发数受临时端口数量限制 → 用多 IP（`SO_REUSEADDR`/多源地址）或连接复用缓解。组播端口天然共享，`SO_REUSEADDR`/`SO_REUSEPORT` 是 UDP 多进程收包的标配（`SO_REUSEPORT` 可做内核级哈希负载均衡）。

### 4.2 最大数据报：65535 与真实上限

| 口径 | 数值 | 推导 |
| :--- | :--- | :--- |
| UDP **长度字段**上限 | $65535\ \text{B}$ | 16 bit |
| UDP 净载理论上限 | $65535 - 8 = 65527\ \text{B}$ | 减 UDP 首部 |
| IPv4 下实际可达上限 | $65535 - 20 - 8 = \mathbf{65507}\ \text{B}$ | IPv4 总长字段同为 16 bit |
| **不分片**（以太网 MTU 1500） | $\mathbf{1472}\ \text{B}$ | $1500 - 20_{\text{IP}} - 8_{\text{UDP}}$ |
| IPv6 不分片（以太网，MTU 1500） | $1452\ \text{B}$ | $1500 - 40_{\text{IPv6}} - 8_{\text{UDP}}$ |
| PPPoE / 隧道 / 802.1Q 环境 | 再减 4 – 68 B | VLAN tag 4 B、PPPoE 8 B、GRE/IPIP 各 4–20 B、IPv6 over IPv4 更多 |

### 4.3 建议数据大小与安全取值

- **以太网 + 无隧道**：单包净载 **1472 B** 恰好填满 MTU；工程上留余量取 **1400 – 1450 B**，或直接按 **1200 B** 设计（IPv6 最小 MTU 即 1280，减去首部后 1232/1200 是跨网安全值）。
- **DNS**：UDP 应答传统限制 512 B 净载（RFC 1035）；EDNS0 建议发 1232 B、收 ≥ 4096 B（RFC 8900 的 PMTU 推荐值）。
- **QUIC**：要求实现至少能收 **1200 B** 的数据报（RFC 9000），典型包大小 1200 – 1252 B，配合 PTU 探测（RFC 8899 DPLPMTUD）。
- **RFC 8900（UDP 缓冲与自动调优 Profile）**：建议接收缓冲 ≥ 2000 数据报、发送上限 4096 B 级别，避免"小缓冲 + 突发"式静默丢包。

$$
L_{\text{app}}^{\max} = \text{MTU} - L_{\text{IP 头}} - 8 \qquad
N_{\text{pkts}} = \left\lceil \frac{M_{\text{app}}}{L_{\text{app}}^{\max}} \right\rceil
$$

> [!CAUTION] 超过 MTU 会发生什么
> IPv4：内核**自动 IP 分片**（DF 未置位时），任一分片丢失 → 整个数据报作废（全或无），且分片放大丢包率、被部分防火墙/负载均衡设备丢弃、无法被中间设备校验。IPv6：**中间路由器不再分片**，DF 语义强制，超大包直接失败并回 ICMPv6 *Packet Too Big*（PMTUD 由发送端负责）。结论：**永远不要依赖 IP 分片承载业务，自己在应用层分片**。

---

## ⚖️ 5. TCP 与 UDP 全面对比

| 维度 | **UDP** | **TCP** |
| :--- | :--- | :--- |
| 连接 | 无连接；无三次握手、无状态、无 FIN 挥手 | 面向连接；`SYN/SYN-ACK/ACK` 建连（1 RTT 后才能发数据），四次挥手释放 |
| 服务模型 | 数据报（保留边界） | 字节流（无边界，需自己 framing） |
| 可靠性 | 不保证；**丢了就是丢了** | 确认 + 超时重传 + 序号 + 重排序，交付或报错 |
| 有序性 | **不保证**（按网络到达顺序上抛） | 严格有序（乱序在接收缓冲重排） |
| 去重 | 不去重（会重复交付） | 序号去重 |
| 流量控制 | 无（内核收缓冲满即丢） | 接收窗口 `rwnd` 通告 |
| 拥塞控制 | 协议本身**无**（应用自行 pacing/CC，如 QUIC/BBR 式） | 内建：慢启动、拥塞避免、快重传、快恢复（CUBIC/BBR） |
| 队头阻塞 | 数据报之间**天然无** HoL | 流语义导致**传输层队头阻塞**（丢 1 包阻塞后续全部字节） |
| 首部 | **8 B 固定**，4 个 16 bit 字段 | **20 B 最小**，选项可到 60 B（MSS/SACK/WS/Timestamp） |
| 协议开销 | 42 B（以太网 + IPv4 + UDP） | 42 B（以太网 + IPv4 + TCP 头 20 B） |
| 首个字节时延 | **0 RTT**（直接发） | 1 RTT（握手）；TFO 可省 1 RTT |
| 吞吐上限 | 无窗口/无 CC 限制，可瞬时打满或撞墙 | 受 $cwnd$、$rwnd$、RTT 限制（BDP 决定） |
| 状态与内存 | 每 socket 一个缓冲队列，几乎无状态 | 每连接维护序列号/重传定时器/窗口/拥塞变量，**易被 SYN flood** |
| 校验和 | 伪首部 16 bit 反码和；IPv4 可选 / IPv6 强制 | 伪首部 16 bit 反码和；**IPv4 也强制** |
| 分片 | 依赖 IP 分片（应避免） | 由 MSS 协商避免 IP 分片（段大小天然受限） |
| 组播/广播 | **支持** | 不支持 |
| NAT/防火墙 | 无连接 → 会话表易超时，需心跳；穿透靠 UDP（STUN/TURN/ICE） | 天然适配（连接明确、可跟踪） |
| 典型端口/业务 | 53 DNS、67/68 DHCP、123 NTP、514 syslog、161 SNMP、69 TFTP、RTP/SRTP、QUIC 443、语音/游戏 | 80/443 HTTP(S)、22 SSH、25 SMTP、1883 [[mqtt]]、8883 |
| 适用判断 | **小包、高频、实时、可容忍丢**、需要组播、要 0 RTT | 大数据完整交付、文件/配置/指令必须不丢不错、需要严格有序 |

> [!TIP] 选型的真实分水岭
> 不是"要不要可靠"，而是**"能不能容忍一次 RTT 的重传等待"**。音视频/控制环路/游戏选 UDP + 前向纠错 + 丢弃过期数据（宁可糊一帧也不要延迟 200 ms）；文件/指令/事务选 TCP 或 **UDP 之上的 QUIC**（把可靠有序交给传输层，同时保留 0-RTT 握手与无队头阻塞的多流）。

---

## 🏗️ 6. 典型应用与"应用层补可靠性"

### 6.1 为什么这些协议选 UDP

| 协议/场景 | 端口 | 选 UDP 的原因 | 可靠性怎么补 |
| :--- | :--: | :--- | :--- |
| **DNS** | 53 | 一问一答小包，建 TCP 太贵；0 RTT 对首屏影响大 | **应用层超时重传**（resolv.conf `timeout:2 attempts:2`）、事务 ID（16 bit）匹配、TCP 兜底用于区域传输/大应答 |
| **DHCP** | 67/68 | 客户端**还没有 IP**，无法完成 TCP 建连；要广播 | 事务 ID（xid）+ 定时重分配（T1/T2）+ 超时重发 DISCOVER，见 [[DHCP]] |
| **NTP** | 123 | 只要**时间戳**，晚到的包没有价值；需要极低抖动 | 不需要可靠——**用最新样本 + 滤波（PLL/IEEE 1588 式）**，重发即新样本 |
| **RTP/SRTP**（音视频、VoIP、WebRTC） | 动态 | 实时性优先，超时重传无意义 | **NACK（RFC 4588）+ FEC（ULPFEC/Opus in-band）+ Jitter Buffer + RTX 重传流 + RTCP 统计回传** |
| **QUIC / HTTP/3** | 443/80 | 用户态实现传输层，快速迭代、0-RTT、消除队头阻塞 | **完整自建**：帧 + 包号 + ACK 帧（含块）+ 恢复算法 + 拥塞控制 + 应用层分片 + 路径 MTU 探测（DPLPMTUD） |
| **游戏网络层**（Unreliable/Reliable UDP、ENet、RakNet、kCP/QUIC 变体） | 动态 | 状态同步要"最新优先"，旧数据可丢弃 | 混合通道：**可靠有序**（事件/伤害）、**可靠无序**、**不可靠有序**（位置快照）三类发送；实体状态用快照 + 增量 + 客户端预测/服务器回滚 |
| **Syslog / SNMP / telemetry** | 514/161 | 高频小包、丢几条日志可接受 | Syslog 靠**多送/冗余**（RFC 5426 式）或换 RFC 5425（TLS over TCP）；SNMP 有确认重发 |
| **TFTP** | 69 | 极简引导场景 | **Lock-and-Ack（停等式 ARQ）** + 505 字节定长块 + 超时重传 |

### 6.2 在 UDP 之上做 ARQ（自动重传请求）

**必备构件清单**：

1. **会话标识**：`session_id` / `connection_id`（防端口复用串包、支持迁移，QUIC 用 8–20 B 的 Connection ID）。
2. **序号与块号**：包级 `packet_number`（单调、参与 ACK）+ 消息级 `msg_id` + `frag_index/frag_total`；序号位数要**大于最大在途窗口**（16 bit 在 BDP 大的链路上会回绕，建议 24/32 bit）。
3. **ACK / SACK**：
   - 累积 ACK（"已收到 ≤N"）+ **SACK 位图/块**（sparse ranges）表示乱序到达；
   - ACK 要**延迟/聚合**（每 N 包或每 10–25 ms），避免 ACK 风暴；
   - 防 ACK 丢失：被确认数据仍留在发送缓冲，直到超时。
4. **重传定时器（RTO）**：按 **RFC 6298** 用 SRTT/RTTVAR 估计（$\alpha=1/8,\beta=1/4$，$RTO=\text{SRTT}+4\cdot\text{RTTVAR}$，下限 200 ms/上限 1 s，指数退避 + 加倍尝试上限）；**Karn 算法**：被重传过的样本不参与 RTT 估计。
5. **快速重传**：收到 N 个（3 个）后向 SACK/重复 ACK 即重发，不等 RTO。
6. **去重与幂等**：接收侧维护"已收窗口 + 已投递位图"，重复包丢弃但**仍要回 ACK**（否则发送方永远重传）。
7. **拥塞/速率控制**：至少做**发送 pacing（均匀发包，避免突发撞队列）+ 丢包退避**；需要公网体面共存就实现 AIMD/BBR 式 CC（QUIC 直接复用成熟实现）。
8. **连接性判定**：心跳 + 空闲超时（NAT UDP 会话常在 **30–120 s** 超时，心跳间隔取半）；`connect()` 后的 `ECONNREFUSED` 可当快速失败信号。

**应用层首部设计参考（≈12 – 16 B，可控）**：

```text
| ver(4) | type(4) | session(24) | msg_id(24) | frag_idx(8) | frag_total(8) | flag(8) | … |  载荷
```

### 6.3 分片与重组（Fragment / Reassembly）要点

- **自己做分片**（每片是一个不超 MTU 的 UDP 数据报），拒绝 IP 分片：单片丢失只重传该片，而不是整个大报文作废。
- 每片必须携带：`session_id`、`msg_id`、`frag_index`、`frag_total`、`payload_len`，**每片独立校验和/CRC**（UDP 校验和 + 可选应用 CRC）。
- 每片首部的 `frag_total` 用于**尽早确定缓冲量与完成度**；建议固定分片大小（$1200 - \text{head}$ 的整数倍）以便 pacing 与统计。
- 重组需要：
  1. **重组超时**（如 2–5 s）：半包到点即整条丢弃并释放内存；
  2. **未完成消息数量/内存上限**（防放大与 DoS：小分片洪水撑爆内存）；
  3. **去重**：同一 `msg_id/frag_index` 重复到达直接覆盖并计数；
  4. **序号回绕与串包防护**：`session_id` 轮换 + 允许的最大窗口；
  5. 乱序分片可用位图（bitset）O(1) 判完成。
- 路径 MTU 用**探测**获得（DPLPMTUD：递增探测包 + 超时降级，QUIC RFC 8899 式），不要盲信 1500；蜂窝/隧道/IPv6-over-IPv4 常把可用载荷压到 1200 甚至更小。

### 6.4 应用层 FEC（前向纠错）

| 方案 | 原理 | 冗余/开销 | 适用 |
| :--- | :--- | :--- | :--- |
| **XOR / 奇偶（even-odd）** | 每 2 包异或多发 1 包 | +50%，只抗**单次**丢包 | 极简单机、链路偶发丢包 |
| **Reed–Solomon $\text{RS}(n+k,\,n)$** | GF($2^8$) 上系统码，任意 $k$ 包丢失可恢复 | 冗余率 $k/n$（语音常 10–30%） | 音视频、文件分发（WebRTC `FlexFEC`、Opus inband FEC） |
| **Raptor / LT（喷泉码）** | 随机度分布编码，收任意 $n(1+\epsilon)$ 个符号可解 | 约 +5–10% | 大对象/批量分发、单向广播链路 |
| **ARQ（重传）** | 请求补发 | 时延 $+$ 1 RTT | 高 RTT 低丢包、允许等待的数据 |
| **HARBD / Hybrid（FEC + NACK）** | FEC 抗小丢包，NACK 补大缺口 | 折中 | **实时媒体与游戏的默认做法** |

**工程规则**：
- FEC 要**跨时间交织（interleave）**：突发丢包（burst）会把连续 $k$ 包一起吞掉，交织后等效随机化，恢复率显著提升。
- FEC **不解决拥塞**：拥塞丢包率高时加冗余只会更拥塞——必须与码率自适应（ABR/GCC）联动。
- 只保护"有价值的新数据"：过期帧、被后续快照覆盖的数据不保护、不重传（**drop instead of resend**）。
- 记录**抖动与单程时延**：实时业务宁可用 FEC 换 1 帧轻微损坏，也不接受 150 ms 重传等待。

---

## ⚠️ 7. 常见坑与排错

### 7.1 丢包静默：最贵的一类 bug

**症状**：`sendto()` 返回成功、对端"有时"收不到、日志什么都不报。
**根因与定位**：

```bash
# 1) 看内核 UDP 计数（最关键：三类错误计数）
netstat -su                      # receive buffer errors / send buffer errors / InCsumErrors
nstat -az | egrep -i 'Udp(RcvbufErrors|SndbufErrors|NoPorts|InCsumErrors|InDatagrams|OutDatagrams)|UdpLite'
ss -u -a -m                      # 每 socket 的 sk_memory / rcv 缓冲占用
# 2) 分片是否发生（分片=丢包放大器）
tcpdump -i eth0 'ip[6:2] & 0x1fff != 0' -nn        # 有 MF/分片偏移的包
# 3) 链路层/驱动丢弃与 offload 计数
ip -s link show eth0 ; ethtool -S eth0 | egrep -i 'drop|err|fifo|no_buffer'
```

**处置顺序**：扩 `SO_RCVBUF` + 开 `SO_RCVLOWAT`/`recvmmsg` 批收 → 单线程收包提速（`recvmmsg` 一次 64 包）→ 发送侧 pacing 削突发 → 多队列/`SO_REUSEPORT` 多进程分担 → 最后才考虑链路排查（Wi-Fi/隧道丢包）。

### 7.2 乱序与重复

- 内核**不重排**：谁先到谁先上抛。表现为"分片号 3 在 1 前面到"、"旧状态覆盖新状态"。
- 排查：`tcpdump -Q in` 与 `SO_TIMESTAMP`（`SO_TIMESTAMPNS`）记录到达时刻，看是否只有某条流乱序（多路径/多队列网卡 RSS 是常见源；可试 `ethtool -L`、`ethtool -k rx-flow-hash`）。
- 处置：应用层序号 + 重排窗口（有序投递）或**语义上幂等 + 以新盖旧**（时间戳/版本号单调，丢弃陈旧包）。

### 7.3 校验和卸载导致的抓包假象

**症状**：Wireshark 满屏 `[bad UDP]` / `Incorrect UDP checksum`，业务却完全正常。
**原因**：TX **checksum offload** 时，内核只把校验和留空/填部分值交给网卡，**抓包点（tcpdump/WinPcap）在网卡之前**，看到的包本来就没有最终校验和；RX 侧 `rx-checksumming: complete` 时内核也不再算，抓包点看到的是占位。

```bash
ethtool -k eth0 | grep -i checksum        # 看 tx-checksumming / rx-checksumming 状态
ethtool -K eth0 tx-checksum-ip-generic off # 临时关卸载，抓"真实"包
ethtool -K lo tx off rx off                # 本机回环抓包最常见假象来源
sudo tcpdump -i eth0 -vv -nn 'udp and port 53'   # 部分版本会标注 [sum ok] / [sum wrong]
```

- Wireshark：`Edit → Preferences → Protocols → UDP → Validate UDP checksum` 关掉，避免噪声掩盖真问题。
- 反向也要警惕：**卸载开启时内核/网卡不会为发出的错误数据算出"看得见"的错**，别把"抓包里校验和正常"当成链路可靠的证据。
- 真校验和错误的判据：接收端计数 `UdpInCsumErrors` 增长（IPv4）/ `Udp6InCsumErrors`（IPv6），或抓包点已在网卡之后（如交换机镜像、`tcpdump` 在 offload 关闭时）且校验和不匹配。

### 7.4 `sendto()` 返回值与错误处理（简）

| 情形 | 表现 | 处理 |
| :--- | :--- | :--- |
| 正常 | 返回**请求发送的字节数**（数据报级，要么全发要么报错） | 仍要 `== len` 判断 |
| 短返回 | 数据报 socket 上返回值 `< len` 意味着**被截断**（少见，多见于 `MSG_MORE`/隧道截断） | 视为错误：记录并重发整包，**不要当成功** |
| `-1 / EINTR` | 被信号打断 | 重试（`do { } while (n<0 && errno==EINTR)`） |
| `-1 / EAGAIN·EWOULDBLOCK` | 非阻塞 + 发送缓冲满（`SndbufErrors` 增长） | 退避重试 / 降速 / 扩 `SO_SNDBUF` / 丢低优先级包 |
| `-1 / EMSGSIZE` | 超过实现上限（$>65535$，或已 `connect()` 的 IPv6/DF 场景超路径 MTU） | 应用层分片；`connect()` 后可读 `IP_RECVERR` 里的 MTU 信息 |
| `-1 / ENOBUFS·ENOMEM` | 内核资源不足 | 限速、扩缓冲、告警 |
| `-1 / EACCES` | 未设 `SO_BROADCAST` 却发广播、SELinux/防火墙拦截 | `setsockopt(SO_BROADCAST)`、查策略 |
| `-1 / ENETUNREACH·EHOSTUNREACH·EPERM` | 路由/接口/策略问题 | 检查路由、链路、防火墙 |
| **异步 ICMP 差错** | 未 `connect()`：通常**完全无感**；已 `connect()`：`ECONNREFUSED` / `EHOSTUNREACH` 在**下一次** `send/recv` 返回（Linux） | 想拿准确差错：`connect()` + `recvmsg(MSG_ERRQUEUE)`（`IP_RECVERR` / `IPV6_RECVERR`）读取 `sock_extended_err`（含 `ee_errno` 与发现的新 `ee_info` MTU） |

```c
for (;;) {
    ssize_t n = sendto(fd, buf, len, MSG_DONTWAIT, dst, dstlen);
    if (n == (ssize_t)len) break;                  /* 完整交付给协议栈 */
    if (n < 0 && errno == EINTR) continue;         /* 信号打断：重试 */
    if (n < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) { /* 缓冲满：退避/丢包 */ continue; }
    handle_send_error(n, errno);                   /* 含 n >= 0 && n != len 的截断场景 */
    break;
}
```

### 7.5 其它高频坑

> [!WARNING] 逐条对照
> - **组播收不到**：忘 `IP_ADD_MEMBERSHIP`、加入网卡选错（`ip_mreqn.imr_ifindex`）、`IP_MULTICAST_IF` 用了默认路由网卡、多网卡/容器网络环境加入 `eth0` 而流量从 `eth1` 来。
> - **组播出不了本机网段**：`IP_MULTICAST_TTL` 默认 **1**（只在链路内）；跨路由需显式设为 >1，且沿途路由器要开 PIM/IGMP。
> - **广播不生效**：`255.255.255.255`（受限广播）不被路由转发；子网定向广播依赖正确掩码，很多边缘设备默认丢弃；容器/mac80211/移动网络常屏蔽广播。
> - **PMTU 黑洞**：ICMP Type 3 Code 4（IPv4）/ Type 2 Packet Too Big（IPv6）被防火墙全拦 → 大包静默消失、小包正常。对策：DPLPMTUD 探测或保守 MTU（1200/1280）。
> - **NAT 会话超时**：UDP 无握手无 FIN，NAT 表项 30–120 s 过期 → 长时间静默后"连不上"。对策：心跳（间隔取超时一半）、`SO_KEEPALIVE` 对 UDP **无效**（不要指望）。
> - **回包发错网卡/源地址**：多地址或多网卡收包后只用 `getsockname()` 决定源 → 用 `IP_PKTINFO`/`recvmsg` 的 `in_pktinfo` 拿到**入接口与目的地址**，同接口回包。
> - **`connect()` 的副作用被忽略**：`connect()` 后源端口固定、非匹配源被丢弃、能收 ICMP 差错；服务端 wildcard + `connect()` 混用会造成 `EISCONN/EADDRINUSE` 困惑。
> - **收包循环里的部分读**：`recvfrom(buf, sizeof buf)` 缓冲小于数据报 → **尾部被截断**且不报错（Linux 行为），要检测"读到的长度是否等于对端发送长度"（应用层带长度字段），或用 `MSG_TRUNC` + `MSG_WAITALL`（不支持则判 `msg_flags & MSG_TRUNC`）。
> - **把 UDP 当"无脑更快"**：不限速的 UDP 会挤爆自家路由器队列，甚至压垮对端 CPU（单核 softirq 收包瓶颈）；上线前压测收包 PPS 与丢包计数。

---

## 🖼️ UDP 首部与收发时序图

![[UDP.excalidraw]]

---

## 🔗 思考与延伸索引

- [[TCP]] — 可靠传输的另一半：状态机、滑动窗口、拥塞控制与队头阻塞（对比 §5）
- [[IP]] — 分片/重组、Identification 回绕、PMTUD 与 ICMP 差错如何决定 UDP 的有效边界
- [[DHCP]] — UDP 端口 67/68 的经典用例：**没有 IP 时如何通信**（广播 + 事务 ID + 客户端状态机）
- [[mqtt]] — 传输层选型对照：MQTT/TCP 的 QoS 语义 vs 用 UDP/QUIC 自建可靠性，"谁负责不丢消息"的分界线

---

## 📚 参考资料

1. **Postel, J.** — *RFC 768: User Datagram Protocol*, USC/Information Sciences Institute, 1980.
2. **Eisenberg, S., Lechlitner, E., Ford, M.** — *RFC 6335: IANA Registry Procedures for the Assignment of Service Names and Port Numbers*, 2011.
3. **Postel, J. (ed.)** — *RFC 1071: Computing the Internet Checksum*, 1988（16 bit 反码求和与实现技巧）.
4. **Braden, R. (ed.)** — *RFC 1122: Requirements for Internet Hosts — Communication Layers*, 1989（§3.3 UDP：校验和实现要求与"可关"条件）.
5. **Deering, S., Hinden, R.** — *RFC 8200: Internet Protocol, Version 6 (IPv6) Specification*, 2017（§8.1：UDP 校验和强制、0 值非法）.
6. **Mankin, A., et al.** — *RFC 1191: Path MTU Discovery*, 1991；**Fairhurst, G., Jones, T., Tüxen, M., Rüngeler, I., Voelker, B.** — *RFC 8899: Packetization Layer Path MTU Discovery for Datagram Transmission*, 2021.
7. **Paxson, V., Allman, M., Chu, J., Scharf, M.** — *RFC 6298: Computing TCP's Retransmission Timer*, 2012（SRTT/RTTVAR/RTO 与 Karn 算法，应用层 ARQ 直接复用）.
8. **Wing, D., Youell, L.** — *RFC 8900: A Profile for UDP Buffer Sizes and Auto-Tuning*, 2021.
9. **IETF AVTCore** — *RFC 4588: RTP Retransmission Payload Format*, 2006；*RFC 8643: Xor Frames for RTP*, 2020（实时业务的 NACK + FEC 组合）.
10. **Iyengar, J., Thomson, M. (eds.)** — *RFC 9000: QUIC: A UDP-Based Multiplexed and Secure Transport*, 2021（含 1200 B 最小数据报、帧/包号/ACK/恢复与 CC 的用户态实现范式）.
11. **Stevens, W. R., Fenner, B., Rudoff, A. M.** — *UNIX Network Programming, Vol. 1, 3rd ed.*, Prentice Hall, 2003（第 18–20 章：UDP 数据报服务、`recvmsg(MSG_ERRQUEUE)`、广播/组播选项）.
12. **Ferguson, D., Huston, G.** — *What Every SysAdmin Should Know About IP Allocations / DNS EDNS0 尺寸实践*, CAIDA, 1998；**Hoffman, P., Bhatia, S., Fu, D., Wing, D.** — *RFC 7766: DNS Transport over TCP*（DNS 的 UDP/TCP 双通道取舍）.
13. **Kurose, J. F., Ross, K. W.** — *计算机网络：自顶向下方法（Computer Networking: A Top-Down Approach）*, 第 7/8 版，第 3 章（运输层：UDP/TCP 原理与 RTP 案例）.
14. **Floyd, K., et al. / Linux man-pages** — *udp(7)*、*ip(7)*、*send(2)*、*recvmsg(2)*、*socket(7)*（`SO_REUSEPORT`、`IP_PKTINFO`、`IP_RECVERR`、`recvmmsg` 行为定义）.
15. **《TCP/IP 详解 卷 1：协议》（TCP/IP Illustrated, Volume 1, 2nd ed.）** — Stevens / Fall，第 10 章（UDP：首部、伪首部、校验和、封装与 MTU 计算）与第 11 章（DNS）.
