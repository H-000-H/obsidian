---
title: IP 协议完整技术手册（IPv4/IPv6）
tags:
  - Network
  - Protocol
  - IP
  - IPv4
  - IPv6
  - Subnetting
  - Fragmentation
status: Completed
created: 2026-09-21
---

# 🌐 IP 协议完整技术手册（IPv4 / IPv6）

> [!INFO] 笔记概览
> - **协议定位**：IP（Internet Protocol，网际协议）是 TCP/IP 体系的**网络层（Layer 3）核心协议**，提供**无连接（Connectionless）、不可靠（Unreliable）、尽力而为（Best-Effort）**的数据报（Datagram）投递；它只负责「把包从源 IP 送到目的 IP」，不保证送达、不保证顺序、不保证不重复。
> - **内容范围**：协议定位 · IPv4 首部逐字段 · 首部校验和 · 分片与重组 · 地址体系与子网计算 · 与 ARP/ICMP 的衔接与转发流程 · IPv6 首部与地址 · Protocol 号与 TTL
> - **推荐路径**：§1 定位 → §2 首部 → §3 校验和 → §4 分片 → §5 地址 → §6 转发 → §7 IPv6 → §8 字段速查

---

## 🧭 1. 协议定位：分层中的 IP

### 1.1 IP 不做的事（不可靠的准确含义）

「不可靠（Unreliable）」**不是**指协议设计得差，而是指 **IP 本身不提供送达确认与恢复机制**：

| IP 不提供 | 由谁提供 |
| :--- | :--- |
| 丢包重传、按序交付 | 传输层 [[TCP]]（序号 + 确认 + 超时重传） |
| 拥塞控制、流量控制 | [[TCP]] 拥塞窗口 / 应用层限速 |
| 端到端完整性校验 | 传输层校验和（TCP/UDP）、应用层 CRC/哈希、TLS |
| 连接建立与拆除 | 传输层（三次挥手）、[[DHCP]]/路由协议等 |
| 带宽与时延保障 | 需 DiffServ/QoS/RSVP 等**额外**机制，best-effort 之外 |
| 送达失败的原因告知 | [[IP]] 只在**本机可判定**时回 ICMP（TTL 超时 / 不可达），且 ICMP 本身也是尽力而为 |

> [!NOTE] 「无连接」的工程含义
> 每个数据报**独立选路**，同一条 TCP 连接的前后两个包可能走完全不同的路径。因此 IP 层没有「会话状态」概念（防火墙/NAT 的连接跟踪是上层加的状态，不是 IP 的）。

### 1.2 与链路层 / 传输层的关系

```text
应用层        │ HTTP / DNS / SSH …           （消息 Message）
─────────────┼──────────────────────────────────────────────
传输层        │ TCP / UDP                     （段 Segment / 数据报 Datagram）
              │ ▲ 提供端口号、端到端可靠性     │
─────────────┼─▼──────────────────────────────┼─────────────
网络层 (IP)   │ IPv4 / IPv6                    （数据报 Datagram / Packet）
              │ ▲ 提供寻址与路由（逻辑地址）   │ 分片只发生在这一层
─────────────┼─▼──────────────────────────────┼─────────────
链路层        │ Ethernet / Wi-Fi / 蜂窝        （帧 Frame）
              │   MAC 地址 = 物理寻址，只在**一跳**内有效
```

| 对比维度 | 链路层（以太网） | 网络层（IP） | 传输层（TCP/UDP） |
| :--- | :--- | :--- | :--- |
| 寻址范围 | 单条链路内（MAC，扁平） | 全网（IP，**层次化**可聚合） | 主机内的进程（端口） |
| 地址是否逐跳改变 | 是（每跳重写二层头） | **否**（源/目的 IP 端到端不变，NAT 除外） | 否（同上） |
| 转发依据 | MAC 表 / 交换机自学习 | **路由表最长前缀匹配** | socket 五元组 |
| 可靠性和顺序 | FCS 检错不重传 | 尽力而为 | TCP 可靠有序 |
| 承载关系 | IP 报文被封装进以太网帧（EtherType `0x0800`/`0x86DD`） | IP 承载 TCP/UDP（Protocol 号区分） | — |

### 1.3 封装与解封装（Encapsulation）

```text
[ 以太网头 14B | IP 头 20B | TCP 头 20B | 应用数据 | FCS 4B ]
   EtherType      Protocol      端口/序号
   =0x0800        =6 (TCP)
```

- **MTU（Maximum Transmission Unit）**：链路层一帧能承载的**净荷上限**，以太网默认 $1500\ \text{B}$；超过它 IP 层必须**分片**或直接丢弃（见 §4）。
- **MSS（Maximum Segment Size）**：TCP 净荷上限，由 PMTUD 反推：$\text{MSS} = \text{MTU} - 40$（IPv4 + 无选项）或 $-60$（IPv6 常见扩展头）。

---

## 📦 2. IPv4 首部逐字段（RFC 791）

### 2.1 总体结构

- **固定首部 20 B**（$IHL = 5$），**最长 60 B**（$IHL = 15$，Options 最多 40 B）。
- 每个字段以 **32 bit（4 字节）为单位对齐**，故首部长度必为 4 的倍数，不足 4 字节用 `Padding`（全 0）填充。
- 位序：**bit 0 为最高位（MSB）**，字节内高位先出、大端（Big-Endian / Network Byte Order）传输。

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver|IHL|  TOS  |  Total Length |       Identification          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Flg|   Fragment Offset (13)   |   TTL     |  Protocol | Header Checksum
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Source Address (32)                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  Destination Address (32)                     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Options (0~360 bit, 可变长) + Padding（可选，无则 IHL=5）     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 2.2 字段逐项说明

| 字段 | 英文 | 位宽 | 字节 | 取值与含义 | 易错点 |
| :--- | :--- | :--: | :--: | :--- | :--- |
| **版本** | Version | 4 bit | 0–3 位 | `4`=IPv4，`6`=IPv6 | 与 IHL 同处首字节，抓包常显示 `IPv4, IHL 5` |
| **首部长度** | IHL（Internet Header Length） | 4 bit | 0–7 位 | 单位是 **4 字节**：最小 `5`(20 B)，最大 `15`(60 B) | 不是字节数！$20 \sim 60\ \text{B}$ 只能是 4 的倍数 |
| **服务类型** | ToS / DS（DSCP 6 bit + ECN 2 bit） | 8 bit | 1 | DSCP 优先级（`EF`=46 语音、`AF31`=26）；ECN：`00` Not-ECT / `01` ECT(1) / `10` ECT(0) / `11` CE | 旧式 Precedence(3b)+TOS(4b)+MBZ 已被 RFC 2474/3168 取代；中间盒常按 DSCP 做 QoS 重标记 |
| **总长度** | Total Length | 16 bit | 2–3 | **首部 + 数据**总字节数，最大 $2^{16}-1 = 65535\ \text{B}$ | 分片后每个片各自填「本片」总长；以太网帧不足 60 B 时填充由链路层补，IP 长度不变 |
| **标识** | Identification | 16 bit | 4–5 | 同一数据报的**所有分片共用同一个 ID**；重组键之一 | 高并发下 16 bit 易翻转复用 → 误重组（Linux 用 IP ID 产生策略可缓解） |
| **标志** | Flags | 3 bit | 6–8 位 | bit0 保留（旧 MF0/DF）；**bit1 = DF**（Don't Fragment）；**bit2 = MF**（More Fragments） | DF=1 且需分片 → 丢弃并回 ICMP Type3 Code4 |
| **片偏移** | Fragment Offset | 13 bit | 6–15 位 | 本片数据相对原数据报起点的偏移，**单位 8 字节** | 最大 $8191 \times 8 = 65528$，配合 16 bit 总长刚好覆盖 65535 |
| **生存时间** | TTL（Time To Live） | 8 bit | 8 | 跳数上限（现为 **Hop Limit** 语义），每过一个路由器 $-1$，为 0 丢弃并回 ICMP Time Exceeded | 最大 255；组播常用 `1` 限制在本网段（`224.0.0.0/24` 要求 TTL=1） |
| **协议号** | Protocol | 8 bit | 9 | 上层协议号，见 §8 | 决定解封装后交给 TCP/UDP/ICMP/GRE 等 |
| **首部校验和** | Header Checksum | 16 bit | 10–11 | 仅对 **IPv4 首部**做 16 bit 反码和取反 | IPv6 取消了该字段；每跳 TTL 变化必须重算 |
| **源地址** | Source Address | 32 bit | 12–15 | 发送方逻辑地址（NAT 可改写） | 端到端「本应」不变 |
| **目的地址** | Destination Address | 32 bit | 16–19 | 接收方逻辑地址（NAT 可改写） | 路由查询用「最长前缀匹配」 |
| **选项** | Options | 0~320 bit | 20+ | `nop`/`eol`/`lsrr`(松散源路由)/`rr`(记录路由)/`ts`(时间戳)/`sec` | 现代网络几乎总被路由器丢弃或视为安全风险；`IHL>5` 少见 |

> [!TIP] 抓包工具里的字段名
> Wireshark 显示 `Flags: 0x40 (Don't Fragment)`，`0x40` = bit1（DF），`0x20` = bit2（MF）。片偏移显示为 `Fragment offset: 185 (1480 bytes)`，括号里才是真实字节偏移。

### 2.3 首部长度速算

$$
\text{Header Length (B)} = IHL \times 4,\qquad IHL \in [5, 15] \Rightarrow 20\text{ B} \le H \le 60\text{ B}
$$

| IHL | 首部长度 | Options 占用 | 场景 |
| :-: | :--: | :--: | :--- |
| 5 | 20 B | 0 B | 绝大多数流量 |
| 6 | 24 B | 4 B | 单个短选项 / IP 记录路由起始 |
| 7–14 | 28–56 B | 8–36 B | 源路由、时间戳 |
| 15 | 60 B | 40 B | 理论最大首部 |

---

## 🧮 3. 首部校验和（Header Checksum）

### 3.1 算法：16 bit 反码求和（One's Complement Sum）

发送端计算：

1. 将 **Header Checksum 字段置 0**；
2. 把首部按 **16 bit 字**（大端）逐个相加，加法产生进位时执行 **end-around carry**（进位回卷，即 $carry$ 加回低位结果）；
3. 对最终 16 bit 结果按位取反，写入 Checksum 字段。

接收/每一跳校验：把**含 Checksum 字段**在内的全部 16 bit 字回卷相加，结果应为 $\texttt{0xFFFF}$（等价于取反后为 $\texttt{0x0000}$），否则**丢弃该数据报**（不回 ICMP，避免 ICMP 风暴与欺骗放大）。

$$
C = \sim\Big(\bigoplus_{i} W_i\Big),\qquad W_i \in \mathbb{Z}_{2^{16}-1},\quad a \oplus b = a + b \bmod (2^{16}-1)
$$

### 3.2 计算示例（20 字节无选项首部）

首部字节（16 进制）：

```text
45 00  00 3C  1C 46  40 00  40 06  [XX XX]  C0 A8  01 42  0A 0B  0C 1D
Ver=4   Len=60 ID     DF,T=0  TTL=64 TCP   校验和  192.168.1.66       10.11.12.29
IHL=5
```

按 16 bit 字求和（校验和字段先置 0）：

| 累加步骤 | 值 | 说明 |
| :--- | :--- | :--- |
| $0\text{x}4500 + 0\text{x}003C$ | $0\text{x}453C$ | |
| $+0\text{x}1C46$ | $0\text{x}6182$ | |
| $+0\text{x}4000$ | $0\text{x}A182$ | |
| $+0\text{x}4006$ | $0\text{x}E188$ | TTL=0x40, Proto=0x06 |
| $+0\text{x}C0A8$ | $0\text{x}1A230$ | **溢出 1**，暂存 32 bit 累加 |
| $+0\text{x}0142$ | $0\text{x}1A372$ | |
| $+0\text{x}0A0B$ | $0\text{x}1AD7D$ | |
| $+0\text{x}0C1D$ | $0\text{x}1B99A$ | |
| 回卷折叠 | $0\text{x}B99A + 0\text{x}1 = 0\text{x}B99B$ | end-around carry |
| 取反 | $\sim 0\text{x}B99B = \texttt{0x4664}$ | **写入 Header Checksum** |

接收端把 `0x4664` 一起加入：$0\text{x}1B99A + 0\text{x}4664 = 0\text{x}1FFFE \to 0\text{x}FFFE + 0\text{x}1 = \texttt{0xFFFF}$ → 校验通过 ✅

### 3.3 每跳增量更新（RFC 1141）

路由器只需把 TTL 减 1，无需重算整个首部。设旧校验和 $C$、旧值 $h$、新值 $h'$（此处 $h=\texttt{0x40}$，$h'=\texttt{0x3F}$）：

$$
C' = \sim\big(\sim C + \sim h + h'\big)
$$

即 $C' = C + (h - h')$ 并处理回卷 —— TTL 每减 1，校验和数值上**加 1**（反码域内的等价操作）。

### 3.4 为什么只校验首部，不校验数据？

| 原因 | 说明 |
| :--- | :--- |
| **每一跳都要校验** | 若覆盖数据，则每个路由器都要读 $65535$ B 级净荷并重算，转发开销不可接受 |
| **首部必然被改** | TTL 每跳 $-1$、Options 可能被修改，首部必须逐跳重算 —— 逐跳检错与逐跳改写刚好匹配 |
| **端到端差错另有归属** | 分层职责：数据完整性由 [[TCP]]/[[UDP]] 校验和（IPv4 下 UDP 校验和可选，IPv6 强制）、链路层 FCS、应用层/TLS 保证 |
| **净荷可能加密** | IPsec ESP / TLS 之后 IP 层无法有意义地校验净荷 |
| **算法太弱** | 16 bit 反码和检错能力有限（可被字节交换、和不变的篡改绕过），不适合做数据完整性 |

> [!WARNING] 校验和不能防篡改
> 反码和是**非密码学**检错码，只能发现偶然比特错误。安全场景请依赖 TCP-AO / IPsec AH-ESP / TLS，见 [[TLS]] 与 [[SSH]]。

> [!TIP] 网卡卸载（Offload）
> Linux `ethtool -k eth0` 中的 `tx-checksum-ip-generic`、`generic-receive-offload`、`large-receive-offload`；开启 TSO/GSO 时由网卡计算校验和并分段，抓包看到的 `bad checksum` 多为伪报错（在 `tcpdump` 前尚未计算）。

---

## ✂️ 4. 分片与重组（Fragmentation & Reassembly）

### 4.1 触发条件与责任边界

- **触发**：待转发的数据报长度 $>$ 出接口 MTU。
- **分片者**：源主机 **或** 路径上任一 MTU 收窄处的路由器（IPv4 允许，IPv6 **禁止路由器分片**）。
- **重组者**：**只有目的主机**（每个片是独立 IP 数据报，可经不同路径乱序到达）。
- 除最后一片外，每片数据长度必须是 **8 的倍数**（因为片偏移单位是 8 B）。

### 4.2 示例：4000 B 数据报按 MTU=1500 分成三片

设 IPv4 首部无选项（$H=20\ \text{B}$），数据报总长 $4000\ \text{B}$，即净荷 $3980\ \text{B}$；出接口 MTU $=1500\ \text{B}$。

每片最大净荷：

$$
L_{\max} = \Big\lfloor \frac{MTU - H}{8} \Big\rfloor \times 8 = \Big\lfloor \frac{1500-20}{8} \Big\rfloor \times 8 = 185 \times 8 = 1480\ \text{B}
$$

| 片 | Identification | DF | **MF** | Fragment Offset 字段 | 真实字节偏移 | 本片净荷 | **Total Length** | 帧占用 |
| :-: | :-: | :-: | :-: | :-: | :--: | :-: | :-: | :--: |
| Frag 1 | `0x1234` | 0 | **1** | **0** | 0 – 1479 | 1480 B | **1500** | 1500 B |
| Frag 2 | `0x1234` | 0 | **1** | **185**（$=1480/8$） | 1480 – 2959 | 1480 B | **1500** | 1500 B |
| Frag 3 | `0x1234` | 0 | **0** | **370**（$=2960/8$） | 2960 – 3979 | 1020 B | **1040** | 1040 B |
| 合计 | 同一 ID | — | — | — | 0 – 3979 | **3980 B** ✅ | $4040 = 4000 + 3 \times 20$ | 开销 +40 B |

要点核对：$1480 + 1480 + 1020 = 3980$（原净荷）✅；三片各自的 TTL / Protocol / 源目的地址相同，仅 `Total Length`、`MF`、`Fragment Offset`、`Header Checksum` 不同。

### 4.3 重组算法（目的主机）

1. **归组键**：`(Source Address, Destination Address, Protocol, Identification)` 四元组 —— 因此不同协议/不同源可复用同一 ID 而不冲突。
2. 按 `Fragment Offset` 排序，记录每片覆盖的字节区间；
3. 收到 `MF = 0` 的片即知**报文总长度**（$\text{offset} \times 8 + \text{TotalLength} - H$），当已收区间恰好覆盖 $[0, \text{总长})$ 时重组完成；
4. 去掉首片首部，把净荷拼接后按 Protocol 交上层。

> [!WARNING] 重组的代价与风险
> - **一片丢失 → 整个数据报作废**（IP 层无选择重传，上层须重传全部数据）；
> - 重组超时（Linux `ipfrag_time`，默认 **30 s**；RFC 791 建议 60 s）未收齐则丢弃已收片，并向源回 ICMP *Time Exceeded, Code 1 (Fragment Reassembly Time Exceeded)*；
> - **分片放大攻击 / Fraggle**：小片洪泛占满重组缓冲；
> - **只有首片含 L4 端口**，防火墙与 IDS 对非首片无法做端口过滤（分片绕过经典成因），需要 `first-frag-only` / 重组式检测；
> - MTU 黑洞（黑洞 PMTU）：ICMP Type3 Code4 被防火墙过滤导致大包静默丢失，表现为「能 ping 通但 HTTPS 打不开」，可用 `iptables --clamp-mss-to-pmtu` 或 `tcp_mtu_probing` 规避。

### 4.4 DF 与路径 MTU 发现（PMTUD, RFC 1191 / RFC 8201）

```text
源主机：发送 DF=1 且长度递增的探测包
   ↓
中间路由器：需要分片但 DF=1 → 丢弃该包，回 ICMP Destination Unreachable
            Type 3, Code 4 "Fragmentation Needed and DF Set"，
            报文内携带下一跳链路 MTU（IPv4 可能需 IP 选项/PMOD，IPv6 由
            ICMPv6 Packet Too Big 直接带 MTU 字段）
   ↓
源主机：把该值写入 PMTU 缓存 → 后续 MSS/分片按 PMTU 收敛
        MSS = PMTU − 40 (IPv4) / − 60 (IPv6 + 扩展头)
```

| 项目 | 说明 |
| :--- | :--- |
| 优点 | 避免分片（分片放大 40 B/片开销、丢片即丢包） |
| 缺点 | 依赖 ICMP 不被过滤；探测包被丢弃时表观为「连接卡死」 |
| 兜底 | PLPMTUD（RFC 8899，Data Packets 探测，QUIC/HBBR 使用）、TCP `mtu_probing` |
| 验证命令 | `ping -M do -s 1472 host`（Linux，$1472 + 28 = 1500$）；Windows `ping -f -l 1472`；`tracepath`/`tracepath6` 内置 PMTU 探测 |

---

## 🔢 5. IPv4 地址体系

### 5.1 分类地址（Classful，历史）与 CIDR（现行）

| 类别 | 前导位 | 地址范围 | 默认掩码 | 网络数 | 每网主机数 | 用途 |
| :-: | :--- | :--- | :--- | :--- | :--- | :--- |
| **A** | `0` | 1.0.0.0 – 127.255.255.255 | /8 | 126（$2^1{-}2$ 扣 0 与 127） | $2^{24}-2$ ≈ 1677 万 | 超大型网络 |
| **B** | `10` | 128.0.0.0 – 191.255.255.255 | /16 | $2^{14}$ = 16384 | $2^{16}-2$ = 65534 | 中型网络 |
| **C** | `110` | 192.0.0.0 – 223.255.255.255 | /24 | $2^{21}$ = 2097152 | $2^{8}-2$ = 254 | 小型网络 |
| **D** | `1110` | 224.0.0.0 – 239.255.255.255 | — | 组播组（无主机概念） | — | 组播 |
| **E** | `1111` | 240.0.0.0 – 255.255.255.254 | — | — | — | 保留/实验（`255.255.255.255` 为受限广播） |

- **CIDR（Classless Inter-Domain Routing，RFC 4632/4291）**：`a.b.c.d/n`，前缀长度 $n$ 与「类别」无关，可任意聚合，是抑制路由表爆炸的关键。
- 掩码必须是**连续 1 后跟连续 0**（VLSM 允许同一网络内用不同 $n$；不等长掩码只在不连续掩码里出现，现代设备普遍不支持）。

### 5.2 私有地址段（RFC 1918，公网路由器必须丢弃）

| 段 | 范围 | 地址数 | 可用主机数 | 典型场景 |
| :--- | :--- | :--: | :--: | :--- |
| `10.0.0.0/8` | 10.0.0.0 – 10.255.255.255 | $2^{24}$ = 16777216 | 16777214 | 大型企业内网、K8s Pod/Service 网段 |
| `172.16.0.0/12` | **172.16.0.0 – 172.31.255.255** | $2^{20}$ = 1048576 | — | Docker 默认网桥、VPN 分配池 |
| `192.168.0.0/16` | 192.168.0.0 – 192.168.255.255 | $2^{16}$ = 65536 | 254/24 | 家用路由器 LAN |
| `169.254.0.0/16` | 链路本地（RFC 3927） | $2^{16}$ | — | APIPA / mDNS，**不可路由** |
| `100.64.0.0/10` | 运营商级 NAT（RFC 6598） | $2^{18}$ | — | CGNAT，隧道/内网穿透常见 |
| `127.0.0.0/8` | 回环 | $2^{24}$ | — | 仅 `127.0.0.1` 常被使用，整段都回环 |

> [!NOTE] 私有 ≠ 不可达
> 私有地址只在**公网**被过滤；同一 VPN/专线内可自由路由。公网 IPv4 枯竭后，`100.64.0.0/10` 与 CGNAT 让「内网套内网」成为常态，排障时要意识到可能有多层 NAT。

### 5.3 特殊地址表（RFC 6890 / RFC 1122 / RFC 919）

| 地址 | 名称 | 可作为源 | 可作为目的 | 含义 |
| :--- | :--- | :-: | :-: | :--- |
| `0.0.0.0` | Unspecified / 全零 | ✅（仅初始化，如 DHCP Discover） | ❌ | 「本机/任意/未知」。路由 `0.0.0.0/0` 表示默认路由 |
| `0.0.0.0/8` | This host on this network | ❌ | ❌ | 现代实现中仅 `0.0.0.0/32` 允许做源（RFC 6890 更新） |
| `127.0.0.0/8` | Loopback | ✅（本机） | ✅ | 不得出现在任何物理链路上 |
| `A.B.C.D`（主机位全 0） | Network Address | ❌ | ❌ | 标识网段本身（CIDR 后允许作为可路由主机地址使用，见注） |
| `A.B.C.255`（主机位全 1） | Subnet-Directed Broadcast | ❌ | ✅ | 子网内广播，路由器默认 `no ip directed-broadcast` |
| `255.255.255.255` | Limited Broadcast | ❌ | ✅ | 只在本链路内传播，**不被路由**（DHCP/ARP 类协议用） |
| `224.0.0.0/24` | Local Network Control（组播） | ❌ | ✅ | `224.0.0.1` 所有主机、`224.0.0.5/6` OSPF、`224.0.0.251` mDNS，要求 TTL=1 |
| `239.0.0.0/8` | Admin Scoped（管理域组播） | ❌ | ✅ | SSDP/DLNA、IPTV 组播 |
| `240.0.0.0/4` | Reserved for Future Use | ❌ | ❌ | 保留（`255.255.255.255` 除外） |

> [!TIP] 「网络地址能不能当主机地址」
> RFC 1122 允许使用主机位全 0 / 全 1 的地址；`/31` 点对点链路按 RFC 3021 只有 2 个可用地址（无网络号与广播号）。但多数家用设备与老式固件仍保留这两个地址，规划时以设备能力为准。

### 5.4 子网计算示例

**例 1：`192.168.10.130/26`**

| 计算项 | 过程 | 结果 |
| :--- | :--- | :--- |
| 子网掩码 | 前 26 位为 1 | `255.255.255.192` |
| 块大小（Block Size） | $256 - 192$ | 64 |
| 网络号 | $\lfloor 130/64\rfloor \times 64 = 128$ | `192.168.10.128` |
| 广播地址 | 网络号 + 63 | `192.168.10.191` |
| 可用主机范围 | 网络号+1 … 广播−1 | `192.168.10.129 – 192.168.10.190` |
| 可用主机数 | $2^{32-26} - 2$ | **62** |
| 该地址所属子网数 | 借 $26-24 = 2$ 位 | 4 个 `/26` |

**例 2：`10.0.6.130/22`**

- 掩码 /22 → `255.255.252.0`，第三字节块大小 $256-252 = 4$；
- $6/4 \to$ 网络基址第三字节 $= 4$ → 网络号 `10.0.4.0/22`，范围 `10.0.4.0 – 10.0.7.255`；
- 广播 `10.0.7.255`，可用主机 $2^{10}-2 = 1022$。

**通用公式**

$$
\text{主机数} = 2^{32-n} - 2,\quad \text{子网数} = 2^{\,n - n_{\text{class}}},\quad
\text{网络号} = \text{IP} \ \&\ \text{Mask}
$$

> [!NOTE] 前缀聚合（Supernetting）
> 把 `192.168.0.0/24 … 192.168.3.0/24` 聚合为 `192.168.0.0/22`；聚合要求起始地址是块大小的整数倍，否则只能拆成多个前缀宣告 —— 这是 BGP 宣告常见坑。

---

## 🔀 6. 与 ARP / ICMP 的衔接与路由器转发流程

### 6.1 IP ↔ ARP（IPv4 必须靠 ARP 拿到下一跳 MAC）

| 要点 | 说明 |
| :--- | :--- |
| 承载 | ARP 是独立链路层协议，EtherType `0x0806`（IPv4 是 `0x0800`，IPv6 `0x86DD`） |
| 解析对象 | **下一跳（Next Hop）的 MAC，不是最终目的的 MAC**：同网段解析目的主机，跨网段解析默认网关 |
| 触发 | 转发前查邻居表；无表项则发 ARP Request（广播 `ff:ff:ff:ff:ff:ff`），超时重试，常见上限 ~10 次后进 `FAILED` |
| 状态 | Linux `ip neigh show`：`REACHABLE / STALE / DELAY / PROBE / FAILED / INCOMPLETE` |
| 免费 ARP（Gratuitous ARP） | 源目 IP 都是自己的 Request/Reply，用于地址冲突检测、VRRP/Keepalived 主备切换时刷新交换机 MAC 表 |
| 代理 ARP（Proxy ARP） | 路由器替他人应答，可做无网关分区，但易致 ARP 表膨胀 |
| 安全 | ARP 无认证 → 中间人/ARP 欺骗；防护用 DAI（Dynamic ARP Inspection）+ DHCP Snooping；IPv6 用 **NDP + SEND(RFC 3971)** 取代 ARP |

### 6.2 IP ↔ ICMP

- ICMPv4 是**用 IP 承载的**（Protocol = `1`），故其差错报文本身也可能丢失 —— 「ICMP 是尽力而为的诊断通道，不能当业务依赖」。
- 与 IP 直接相关的 ICMP 类型：

| Type / Code | 名称 | 触发场景（与 IP 层的关系） |
| :-: | :--- | :--- |
| 3 / 0 | Network Unreachable | 路由表无匹配（`ip route` 黑洞 / `no route to host`） |
| 3 / 1 | Host Unreachable | ARP 解析不到下一跳 |
| **3 / 4** | **Fragmentation Needed and DF Set** | **PMTUD 核心**（§4.4） |
| 3 / 13 | Communication Administratively Prohibited | ACL/防火墙明确 reject |
| 8 / 0 · 0 / 0 | Echo Request / Reply | `ping` |
| **11 / 0** | Time Exceeded (TTL=0) | **traceroute 原理**（§8.2） |
| 11 / 1 | Time Exceeded (Reassembly) | 分片重组超时 |
| 12 / 0 | Bad Option | 首部选项无法处理 |
| 5-x | Redirect | 网关告知「有更优下一跳」（常被安全策略关闭） |

### 6.3 路由器转发一个 IPv4 数据报的完整流程

```text
① 入接口收到帧 → 校验 FCS → 按 EtherType 上送 IPv4
② 校验 IP 首部 Checksum（≠0xFFFF 则丢弃，不回 ICMP）
③ 目的 IP 是否是本机？（本机 IP / 广播 / 命中的组播组）→ 是则上送本机协议栈，否则继续转发
④ 最长前缀匹配查路由表（ip route / FIB）→ 得到 出接口 + 下一跳(网关)
   无匹配 → 丢弃并回 ICMP Type3 Code0（源为不可达组播/广播地址时不回）
⑤ 安全与策略检查：ACL / uRPF / 防火墙 / 是否禁止转发广播、源路由选项
⑥ TTL ← TTL − 1；若 TTL == 0 → 丢弃并回 ICMP Type11 Code0（traceroute 就靠这一步）
⑦ MTU 检查：Length ≤ 出接口 MTU？
     ├─ 是 → 不分片
     └─ 否 → DF=1：丢弃 + 回 ICMP Type3 Code4（携带 MTU）
              DF=0：按 §4.2 分片，每片各自重新封装
⑧ 首部有变 → 重算（或增量更新）Header Checksum
⑨ 重写二层头：src MAC = 出接口 MAC，dst MAC = 下一跳 MAC（ARP/NDP 解析）
⑩ 出队 / 队列调度（QoS、WFQ、Tail Drop/RED/WRED）→ 发送；同时统计与采样（NetFlow/sFlow）
```

> [!WARNING] 常见「转发不通」定位顺序
> `ip route get <dst>`（选路）→ `ip neigh`（ARP/NDP）→ `ping -M do -s <len>`（MTU/DF）→ `tcpdump -i <if> icmp`（看 ICMP 差错被谁吞了）→ ACL/安全组。转发路径上源/目的 IP 不变，但 **MAC 每跳都变**，所以抓包只在入口和出口各抓一次即可判断。

---

## 🆕 7. IPv6（RFC 8200 / RFC 4291）

### 7.1 固定 40 字节首部

IPv6 把可变长选项移出首部（改为 **Next Header 串成的扩展首部链**），使路由器只需解析定长首部，转发更快：

| 字段 | 英文 | 位宽 | 字节偏移 | 说明 |
| :--- | :--- | :--: | :--: | :--- |
| 版本 | Version | 4 bit | 0（高 4 位） | 固定 `6` |
| 流量类别 | Traffic Class | 8 bit | 1 | DSCP(6)+ECN(2)，与 IPv4 ToS 语义一致 |
| 流标签 | Flow Label | 20 bit | 1(低 4 位)–3 | 源请求的「流」级 QoS/熵源（`ip6tables`/硬件流表可用） |
| 净荷长度 | Payload Length | 16 bit | 4–5 | **仅净荷**（含扩展首部），**不含 40 B 固定首部**；Jumbogram 用 HBH Option 扩展为 32 bit（置 0 + `Payload Option`） |
| 下一首部 | Next Header | 8 bit | 6 | 相当于 IPv4 Protocol；也用于指示**扩展首部链** |
| 跳数限制 | Hop Limit | 8 bit | 7 | 等价 TTL，**逐跳减 1**（不再有分片语义） |
| 源地址 | Source Address | 128 bit | 8–23 | IPv6 地址 |
| 目的地址 | Destination Address | 128 bit | 24–39 | IPv6 地址 |

$$
4 + 8 + 20 + 16 + 8 + 8 + 128 + 128 = 320\ \text{bit} = 40\ \text{B（固定，无选项）}
$$

**没有**：Identification、Flags、Fragment Offset、Header Checksum（IPv6 逐跳校验和取消，依赖链路层 FCS + 传输层校验和）。

常用扩展首部（Next Header 值）：`0` Hop-by-Hop、`43` Routing、`44` Fragment、`60` Destination Options、`51` AH、`50` ESP、`59` No Next Header。其中**只有源主机能给 Fragment 头**（`Offset`、`M` 标志、`ID` 在此）。

### 7.2 地址表示与压缩规则

| 规则 | 示例 |
| :--- | :--- |
| 8 组 × 16 bit，冒号分隔，十六进制 | `2001:0db8:0000:0000:0000:0000:0000:0001` |
| **省略前导零** | `2001:db8:0:0:0:0:0:1` |
| **`::` 压缩连续全零段**（每个地址**只能出现一次**） | `2001:db8::1` |
| 前缀记法 `/n` | `2001:db8::/32`（站点前缀）、`2001:db8:1::/64`（子网） |
| 环回 / 未指定 | `::1` / `::` |
| 链路本地（强制自动生成） | `fe80::/10`，如 `fe80::21a:2bff:fe35:1234` |
| 唯一本地地址 ULA | `fc00::/7`（实际 `fd00::/8`）≈ IPv4 私有限定 |
| 全局单播 GUA | `2000::/3`（当前主要 `2001::/16`、`2400::/12` 等） |
| 组播 | `ff00::/8`：`ff02::1` 所有节点、`ff02::2` 所有路由器、`ff02::1:ffXX:XXXX` 被请求节点（替代广播） |
| 内嵌 IPv4 | `::ffff:192.168.1.1`（v4-mapped）、`64:ff9b::/96`（NAT64 Well-Known Prefix）、`2002::/16`（6to4） |
| 接口 ID 派生 | EUI-64（MAC 插 `ff:fe` 并翻转 U/L 位）或 `stable-privacy`/RFC 7217 生成地址（防跟踪） |

> [!CAUTION] `::` 只能一次
> `2001::db8::1` 非法（无法确定各零段长度）。含 IPv4 的点分尾段只允许出现在最后 32 bit，如 `::1.2.3.4`。

### 7.3 IPv4 与 IPv6 关键差异对比

| 维度 | IPv4 | IPv6 | 影响 |
| :--- | :--- | :--- | :--- |
| 地址长度 | 32 bit（$\approx 4.3\times10^9$） | 128 bit（$3.4\times10^{38}$） | 无需 NAT、端到端可达回归 |
| 首部长度 | 20–60 B 可变 | **固定 40 B** | 转发解析更快 |
| 首部校验和 | 有（每跳重算） | **无** | 省 CPU，端到端差错交给上层 |
| 路由器分片 | **允许** | **禁止**（仅源主机用 Fragment 扩展头） | 中间设备无重组负担；PMTUD 成强制设计前提 |
| 分片信息位置 | 首部固定字段 | 端到端 Fragment 扩展头 | 路径 MTU 更稳 |
| 协议识别 | Protocol 字段 | Next Header 链 | 可扩展、可加密（ESP 之后不可见） |
| 广播 | `255.255.255.255`、子网定向广播 | **无广播**，全部组播/链路本地 | 二层泛洪大幅减少，交换机 MAC 表更稳定 |
| 邻居发现 | ARP（EtherType 0x0806） | **NDP**（ICMPv6 Type 133/135/136 等，组播目标） | 依赖组播可用；ND 表可加密（SEND） |
| 配置 | 手动 / [[DHCP]] | SLAAC（RA + EUI-64/隐私地址）、DHCPv6（状态/无状态）、前缀委派 PD | 插上就能用；排障要看 RA 与 `accept_ra` |
| 组播支持 | 可选、部署差 | 必须支持，SSM/PIM 基础 | 转发的组播必须放行 `ff02::/16`、ICMPv6 |
| 选项 | Options 在首部内 | 扩展首部链（HBH/Routing/Dest/Frag） | 安全设备常因无法解析而拦截 IPv6 |
| 中间盒/NAT 可观测性 | 深度包检测成熟 | 加密/扩展头穿透，NAT64/ALG 复杂 | IPv6 只放行不检测 = 风险 |
| 常见 MTU | 1500 | **1280 最小必须**，链路仍 1500 | 隧道开销（6in4 +20、6to4 +20、Teredo +40）易触发 PMTUD 黑洞 |
| 安全性 | IPsec 可选 | 协议设计要求支持 IPsec（实际部署少） | 仍需 TLS 提供端到端加密 |

### 7.4 过渡机制（一句话级）

- **双栈（Dual Stack）**：同一主机同时跑 v4/v6，按 Happy Eyeballs 优先 v6 —— 最主流的部署形态。
- **NAT64 / DNS64**：DNS64 把只有 A 记录的目标合成 AAAA（`64:ff9b::/96`），NAT64 做协议转换，让纯 v6 客户端访问 v4 服务。
- **464XLAT（CLAT+464）**：在 64/464 场景为应用层（LTE/蜂窝、IPv6-only 网络）补上 v4 通路，避免 APP 逐个改造。
- **6in4 / 6to4 / ISATAP（配置化隧道）**：把 IPv6 包封进 IPv4（Protocol `41`）穿越 v4 骨干；6to4 用 `2002::/16` 自动嵌入 v4 地址，已不推荐。
- **Teredo**：IPv6 封在 **UDP** 里以穿过 NAT（NAT-T 友好），现多已停用。
- **DS-Lite**：用户侧发 IPv4，运营商城域网封装进 IPv6 隧道后集中做 CGNAT —— 「v6 只当承载，用户仍看到 v4」。

---

## 🏷️ 8. Protocol 号、TTL 与 traceroute

### 8.1 常用 IPv4 Protocol 号（亦作 IPv6 Next Header）

| 号 | 协议 | 说明 / 常见位置 |
| :-: | :--- | :--- |
| **1** | **ICMP** | `ping`、差错报文、traceroute(UDP 模式) 的返回值 |
| 2 | IGMP | 组播组成员报告（IPv6 由 MLD 取代，Next Header `0` + ICMPv6 Type 130/131/132） |
| 4 | IP-in-IP | 协议 4 隧道，`ip tunnel` / IPIP、部分 VPN |
| **6** | **TCP** | 见 [[TCP]]；HTTP/HTTPS/SSH 的承载 |
| 8 | EGP | 外部网关协议（历史遗留） |
| **17** | **UDP** | 见 [[UDP]]；DNS/DHCP/VoIP/QUIC |
| 41 | IPv6 | 6in4/6to4/ISATAP 隧道（IPv6 over IPv4） |
| **47** | **GRE** | 通用路由封装（VPN 接口、SD-WAN Overlay）；注意 GRE 无 NAT 转换友好性 |
| 50 | ESP | IPsec 加密载荷（协议 50，端口概念消失，NAT-T 需 UDP 4500 封装） |
| 51 | AH | IPsec 认证头（覆盖首部，遇 NAT 即失效） |
| **58** | **IPv6-Headers / ICMPv6** | IPv6 语境下 `58` 指 **ICMPv6**（ND/RA/NS/NA/Packet Too Big） |
| 59 | No Next Header | IPv6 扩展链末端，其后无上层协议（或为 ESP 载荷） |
| 88 / 89 | EIGRP / OSPF | 动态路由协议直接跑在 IP 上（OSPF 组播 `224.0.0.5/6`） |
| 112 | VRRP | 网关冗余（组播 `224.0.0.18`） |
| 132 | SCTP | 多流传输，Diameter/SIP 底层 |

> [!NOTE] 完整清单
> IANA *Protocol Numbers* 注册表：https://www.iana.org/assignments/protocol-numbers/ 。防火墙放行政策时，隧道/路由类协议（41/47/50/51/58/89）最容易被漏放，导致「ping 通但不通业务」或 BGP/OSPF 邻居起不来。

### 8.2 TTL 的两个用途小注

1. **防环 + 生存期**：$TTL \ge$ 网络最大跳数才不会误丢；`tracert` 依赖逐跳递减。
2. **指纹识别（OS Fingerprinting）**：不同系统默认 TTL 不同，可粗略判断设备类型：

| 默认 TTL | 常见来源 |
| :-: | :--- |
| 64 | Linux、macOS/iOS、多数网络设备 |
| 128 | Windows（`DefaultTTL` 注册表可改） |
| 255 | Cisco IOS/路由器、部分 BSD、嵌入式设备 |
| 60 / 30 | 老式 UNIX、某些 AP/摄像头 |
| 2 / 3 | 组播协议报文（如 `224.0.0.x` 强制 TTL=1/2） |

**traceroute 原理**：源主机依次以 $TTL = 1, 2, 3, \dots$ 发包；每跳路由器把 TTL 减 1，变为 0 时丢弃并回 **ICMP Type 11 Code 0**，源据此打印该跳地址与 RTT；当包到达目的主机时：

| 模式 | 发包 | 目的主机的应答 | 默认端口/字段 |
| :--- | :--- | :--- | :--- |
| UDP 模式（Linux 默认） | UDP 高端口（33434 起） | ICMP **Type 3 Code 3**（Port Unreachable） | 目的端口避开服务 |
| ICMP Echo 模式（Windows `tracert`） | Echo Request | Echo Reply（不报 RTT 跳） | Type 8 |
| TCP SYN 模式（`traceroute -T -p 443`） | TCP SYN:443 | SYN/ACK 或 RST → 判定可达 | 用于穿防火墙只开 443 的路径 |
| IPv6（`traceroute6`/`tracepath6`） | HBH Hop Limit 递增 | ICMPv6 **Type 2**（Packet Too Big）/Type 3 | 同时探 PMTU |

> [!TIP] 中间设备不回 ICMP 时
> 出现 `* * *` 多为：① 该跳出于安全不生成/限速 ICMP；② 回程路由缺失。可换 `mtr`（持续采样）、`traceroute -T`（TCP 模式）、或看 `tracepath` 的 PMTU 结论。

---

## 🖼️ IPv4 首部与分片时序图

![[IP.excalidraw]]

---

## 🔗 思考与延伸索引

- [[TCP]] —— IP 之上的可靠字节流：序号/确认/重传如何补上 IP 的「不可靠」，MSS 与 PMTU 的换算
- [[UDP]] —— 与 IP 同样尽力而为的无连接层：IPv4 校验和可选、IPv6 强制；QUIC 在 UDP 上做可靠传输
- [[DHCP]] —— 用 UDP + 广播拿到 IP/掩码/网关：与本篇 `0.0.0.0` 源地址、广播地址、`ip route` 默认网关直接呼应
- [[HTTP]] —— 应用层如何受 MTU/PMTUD 黑洞与分片影响（「能 ping 通但网页打不开」）
- [[TLS]] / [[SSH]] —— 加密后 IP 层只见首部，DPI 与分片绕过策略的边界
- [[ICMP]] —— 差错与诊断通道：Type/Code 全表与限速策略
- [[ARP]] —— 下一跳 MAC 解析、代理 ARP、ARP 欺骗与防御
- [[Modbus]] —— 现场总线上 Modbus TCP（端口 502）在 IP 层分片/PMTU 的取舍

---

## 📚 参考资料

1. **Postel, J.** — *RFC 791: Internet Protocol*, 1981（IPv4 首部、分片、TTL、校验和的规范原文）. https://www.rfc-editor.org/rfc/rfc791
2. **Deering, S., Hinden, R.** — *RFC 4291: IP Version 6 Addressing Architecture*, 2006. https://www.rfc-editor.org/rfc/rfc4291
3. **Hinden, R., Deering, S.** — *RFC 8200: Internet Protocol, Version 6 (IPv6) Specification*, 2017（取代 RFC 2460）. https://www.rfc-editor.org/rfc/rfc8200
4. **Mogul, J., Kent, S.** — *RFC 1141: Stream Throughput of Internet Protocols*（IP 首部校验和增量更新算法）. https://www.rfc-editor.org/rfc/rfc1141
5. **McCann, J., Gupta, S., Mogul, J.** — *RFC 1191: Path MTU Discovery*. https://www.rfc-editor.org/rfc/rfc1191
6. **Touch, J. et al.** — *RFC 8899: Packetization Layer Path MTU Discovery for Datagram Transports*, 2020.
7. **Huitema, C.** — *RFC 4632: Classless Inter-Domain Routing (CIDR)*；*RFC 1918: Address Allocation for Private Internets*；*RFC 6890: Special-Purpose IP Address Registries*.
8. **Cotton, M. et al.** — *RFC 5237: IANA Protocol Number Registry Update*；IANA *Protocol Numbers* 注册表.
9. **Ferguson, P., Huston, G.** — *DNS and DHCP Security / The Design and Implementation of the Internet*（CIDR 与地址耗尽分析）.
10. **Stevens, W. R.** — 《TCP/IP 详解 卷 1：协议》（*TCP/IP Illustrated, Vol.1*, 2nd ed., 2023）—— 第 3–5 章 IP 首部/分片/路由、第 13 章 ARP、第 6 章 ICMP.
11. **Comer, D.** — 《用 TCP/IP 网际互联》（*Internetworking with TCP/IP Vol. I*）—— 子网划分与地址聚合的系统化推导.
12. **Linux 内核文档** — *Documentation/networking/ip-sysctl.rst*（`ipfrag_time`、`ip_no_pmtu_disc`、`accept_ra`、MSS clamping）；`iproute2` man pages（`ip-route`、`ip-neighbour`）.
13. **Cisco** — *Configuring IP Services / Path MTU Discovery & TCP MSS Adjustment* 应用笔记.
