---
title: DHCP 协议完整技术手册
tags:
  - Network
  - Protocol
  - DHCP
  - UDP
  - BOOTP
  - Troubleshooting
status: Completed
---

# 🔗 DHCP 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：DHCP（Dynamic Host Configuration Protocol，动态主机配置协议）是承载在 UDP 之上的应用层协议，客户端用**一条广播报文**换回"一个 IP + 整套网络参数（掩码/网关/DNS/租期…）"。它是 BOOTP 的向后兼容演进版，核心增量是**租期（Lease）**与**选项体系（Options）**。
> - **内容范围**：协议定位与承载 · DORA 全流程与附加状态 · 报文逐字段 · Options 重点表 · 地址池与绑定 · 中继（GIADDR） · 多网卡 metric · APIPA · 安全 · 排错
> - **推荐路径**：§1 定位 → §2 DORA（先懂流程）→ §3/§4 报文与选项（再抠细节）→ §5–§7 服务端与客户端行为 → §8–§10 边界场景、安全与排错

---

## 📡 1. 协议定位与承载

### 1.1 从 BOOTP 到 DHCP

DHCP 没有重新设计报文，而是**直接沿用 BOOTP 的包格式与端口**（RFC 2131 §1/§8），只在 options 区扩展语义：

| 维度    | BOOTP（RFC 951/1542）        | DHCP（RFC 2131/2132）             |
| :---- | :------------------------- | :------------------------------ |
| 地址来源  | 服务器数据库中**逐台静态绑定** MAC → IP | **地址池动态分配** + 可选保留（reservation） |
| 地址有效期 | 永久（直到人工改动）                 | **有租期**（option 51），到期续租/回收      |
| 消息细分  | 只有 request / reply 两种      | option 53 细分出 8+ 种消息类型          |
| 参数下发  | 仅 vendor 扩展区（无标准语义）        | 标准化选项表（RFC 2132 及其后大量 RFC 增补）   |
| 端口    | 67 / 68                    | 67 / 68（同一套，中继与服务器需同时兼容 BOOTP）  |
| 典型场景  | 早期无盘工作站                    | 现代所有即插即用终端、PXE、VoIP、IoT         |

> [!TIP] 为什么现在还提 BOOTP
> 报文里 `op`/`htype`/`hlen`/`hops`/`siaddr`/`file` 等字段都是 BOOTP 时代的遗产（为 TFTP 引导服务）。看到"PXE 启动拿不到 boot file"这类问题时，要回到 BOOTP 语义去理解 `siaddr`（下一跳引导服务器）与 `file`（引导文件名）。

### 1.2 UDP 承载与端口约定

| 角色          |  源端口   |  目的端口  | 说明                               |
| :---------- | :----: | :----: | :------------------------------- |
| 客户端 → 服务器   | **68** | **67** | 客户端必须绑定 68；服务器监听 67              |
| 服务器 → 客户端   | **67** | **68** | 单播或广播回包                          |
| 中继 ↔ 服务器    |   67   |   67   | 中继与服务器**同端口通信**，靠 giaddr/hops 区分 |
| BOOTP 中继（旧） |   67   |   67   | 与上同                              |

- 端口写死（Well-Known）的意义：客户端在**完全没有网络配置**时也要知道往哪个端口发；防火墙/交换机 ACL 也能无状态放行。
- **RFC 6842** 明确要求：客户端的第一个 DISCOVER **必须**从源端口 68 发出（早期某些实现用 ephemeral port，服务器需兼容）。
- 因为 UDP 无重传、无拥塞控制（见 [[UDP]]），DHCP 自己在客户端侧做**超时 + 指数退避重传**（见 §2.7 INIT-BACKOFF）。
- DHCPv6 完全不同的设计：使用 UDP **546/547**、链路本地地址 + 组播 `ff02::1:2`/`ff02::1:3`，无 0.0.0.0 广播，不在本文范围。

### 1.3 广播 / 单播规则（关键考点）

| 报文                                    | 源 IP : 端口     | 目的 IP : 端口               | 二层目的                |
| :------------------------------------ | :------------ | :----------------------- | :------------------ |
| DISCOVER / REQUEST（选路阶段）              | `0.0.0.0:68`  | `255.255.255.255:67`     | `FF:FF:FF:FF:FF:FF` |
| OFFER / ACK（B=1 或 giaddr=0 且客户端不能收单播） | `serverIP:67` | `255.255.255.255:68`     | `FF:FF:FF:FF:FF:FF` |
| OFFER / ACK（B=0 且与客户端同二层域）            | `serverIP:67` | `yiaddr:68`              | 客户端单播 MAC           |
| OFFER / ACK（**giaddr ≠ 0，中继场景**）      | `serverIP:67` | `giaddr:67`（单播给中继）       | 中继 MAC              |
| RENEW（T1 后）                           | `自己的 IP:68`   | `服务器 IP:67`（单播）          | 网关/服务器 MAC          |
| REBIND（T2 后）                          | `自己的 IP:68`   | `255.255.255.255:67`（广播） | 广播                  |

**`flags` 的 B 位（bit0）语义**：

- `B = 1`：服务器**必须**用广播（`255.255.255.255`）回包。老式客户端在拿到地址前无法正确处理发给"自己新地址"的单播帧（ARP 还没建立 / 协议栈未就绪），所以置 1。
- `B = 0`：客户端声明"我能收单播"，服务器**可以**单播到 `yiaddr:68`。
- 强制覆盖条件：只要 `giaddr ≠ 0`，服务器一律**单播给中继的 67 端口**，B 位被忽略（广播域不能跨路由器）。
- 现代 Windows / Linux（systemd-networkd、dhcpcd、dhclient）默认 `B = 0`，但两者都能收；遇到"服务器回单播、客户端收不到"的现象，多为中间设备把未知单播丢弃，或部分 IoT 轻量协议栈只实现了广播收包。

---

## 🔄 2. DORA 四阶段详解

### 2.1 阶段总览

```text
Client                                     DHCP Server
  │  ① DISCOVER   opt53=1   bcast              │
  │ ─────────────────────────────────────────► │  选地址、挂 pending binding
  │  ② OFFER      opt53=2   yiaddr=提议地址     │
  │ ◄───────────────────────────────────────── │
  │  ③ REQUEST    opt53=3   opt50+opt54        │
  │ ─────────────────────────────────────────► │  正式确认、写 binding DB
  │  ④ ACK        opt53=5   opt51/58/59        │
  │ ◄───────────────────────────────────────── │
  │        （T1 ≈ 50% 租期 → RENEW 单播）        │
  │        （T2 ≈ 87.5% 租期 → REBIND 广播）     │
```

| 阶段 | 消息（opt 53） | 客户端状态 | 服务器动作 | 关键字段 |
| :--- | :--: | :--- | :--- | :--- |
| ① Discover | 1 | INIT / SELECTING | 收到后从匹配作用域的池里挑一个**未过期**地址，暂不正式确认 | `xid` 随机、`ciaddr=0`、`chaddr=MAC`、opt 55 参数请求列表 |
| ② Offer | 2 | SELECTING | 单播/广播回一条提议，`yiaddr` = 提议地址，opt 54 = 服务器 ID | `xid` 回填、opt 54、opt 1/3/6/51 |
| ③ Request | 3 | REQUESTING | **广播**（故意让所有服务器听到，落选者释放其 Offer） | opt 50（想要的地址）、opt 54（选中的服务器） |
| ④ ACK / NAK | 5 / 6 | BOUND / BACKOFF | 通过 → 写绑定与租约；拒绝（地址不可用/不在该网段）→ NAK | opt 51/58/59 及全部下发参数 |

### 2.2 ① DISCOVER：找服务器

- 客户端**还没有 IP**，只能用 `0.0.0.0:68 → 255.255.255.255:67`；ARP 层同时先做接口绑定。
- `xid` 由客户端随机生成，**本轮 DORA 四个包保持一致**，客户端据此丢弃其它会话的应答。
- 携带 opt 55（Parameter Request List）告诉服务器"我只需要这几项"，减少无谓下发；opt 60/61 表明厂商类别与客户端标识。
- 服务器侧行为：`secs`/时间窗内可配置 `delay`（多台服务器错峰，避免同时回 Offer）；对已知不可重复使用的地址（`conflict` 标记、被保留、正在 failover 配对）跳过。

### 2.3 ② OFFER：服务器报价

- 一个客户端可能收到**多条 OFFER**（多台服务器），客户端策略通常是**取第一条**（Windows/dhclient 默认），其余 Offer 由对应服务器在短超时后自动回收，不占用租约。
- `yiaddr` 填提议地址；`siaddr`/`file` 在 PXE 场景填引导服务器与文件名。
- OFFER 必须带 **opt 54 Server Identifier**（服务器的 IP，多网卡服务器用它表明"用哪个地址与我通信"）——这是后续 REQUEST 定向的依据。

### 2.4 ③ REQUEST：确认选型

- **广播发送**（不是单播给选中的服务器）：让落选的服务器立即释放各自的 Offer，缩短地址占用。
- opt 50（Requested IP Address）= 想要的地址；opt 54 = 选中的服务器 ID。两个都带是"我确实要跟您成交"的语义，缺一不可，否则服务器无法判断自己是否被选中。
- 服务器返回 **ACK**（正式建约，写入 lease DB）或 **NAK**（地址不能用，客户端回到 INIT-SELECTING 重来）。

### 2.5 ④ ACK：参数生效

- 客户端拿到 `yiaddr` + opt 1/3/6/51/58/59 等，配置接口；Windows/Linux 通常会先用 **ARP 探测**（gratuitous ARP / ARP check，RFC 5227 思路）验证地址未被占用，冲突则发 **DECLINE**（opt 53=4）并重新 DISCOVER。
- ACK 之后进入 **BOUND**，同时启动两个定时器 T1/T2。

### 2.6 为什么客户端源 IP 用 `0.0.0.0`

1. **协议层**：客户端此时**尚未获得地址**，`0.0.0.0` 表示"未指定/未知"，是唯一语义正确的源地址；不能用别的地址，否则可能被当成合法主机而破坏服务器/防火墙判断。
2. **实现层**：协议栈在接口无地址时无法完成路由与 ARP，因此 DHCP 客户端进程通过原始套接字 / `AF_PACKET`（Linux）或专属的 AFD/DHCPCLNT 通道（Windows）收发，源 IP 固定 `0.0.0.0`、源端口固定 68。
3. **网络层**：目的地址 `255.255.255.255` 是**有限广播（limited broadcast）**，路由器默认**不转发**，所以 DHCP 流量不会越三层域——跨网段必须靠 §6 的中继。
4. **回包方向**：服务器不能直接单播给"还不存在的地址"，除非 B=0 且客户端确认自己能收单播；这也是 B 位存在的原因。

### 2.7 附加流程：Init-Reboot、Renew、Rebind、Release、Inform、Decline

| 流程 | 触发时机 | 消息与关键字段 | 服务端结果 |
| :--- | :--- | :--- | :--- |
| **INIT-REBOOT**（快速复用旧地址） | 客户端重启，本地缓存的地址+租约还没到期 | 广播 **REQUEST**：`opt 50 = 旧地址`、**不带** opt 54 | 服务器认为该地址仍有效 → **ACK**（续租）；地址不该在此网段/已被占用 → **NAK** → 客户端退回 INIT-SELECTING 走完整 DORA |
| **RENEW（T1 时刻）** | 租期用掉 **50%** | **单播** REQUEST 给"当初给我地址的那台服务器"：`ciaddr = 自己的 IP`，**不带** opt 50 / opt 54 | 服务器直接 **ACK** 刷新租期（同一地址）；无响应 → 什么都不丢，继续用旧地址，进入 REBIND |
| **REBIND（T2 时刻）** | 租期用掉 **87.5%**（原始租期 100% 前） | **广播** REQUEST（`ciaddr` 保持自己的地址），任意服务器都能应答 | 换一台服务器也可能**换地址**；NAK → 立即 INIT-SELECTING；仍无响应 → 租期到点后地址失效，回到 DISCOVER |
| **RELEASE** | 客户端主动下线（关网卡、`ipconfig /release`） | 广播 RELEASE（opt 53=7），`ciaddr` = 自己的地址 | 服务器把绑定标记释放，地址回池 |
| **INFORM** | 地址已由静态/SLAAC/PPPoE 等方式确定，**只要参数** | 广播 INFORM（opt 53=8），`ciaddr` = 自己的地址 | 服务器回 **ACK** 但**不分配地址**（`yiaddr=0`） |
| **DECLINE** | ARP 检测到 ACK 给下来的地址已被占用（DAD 冲突） | 广播 DECLINE（opt 53=4），`ciaddr` = 冲突地址 | 服务器把该地址标为 `conflict`，一段时间内**冻结不再分配**（ISC 默认约 1 小时，管理员需人工检查） |

$$
T_1 = 0.5 \times \text{lease} \qquad T_2 = 0.875 \times \text{lease}
$$

> [!IMPORTANT] 默认值可被覆盖
> T1/T2 由 **option 58 / 59** 显式下发（单位秒）。不下发时客户端按上式自算；Windows 客户端注册表 `Dhcp\{GUID}\Parameters` 也支持固定值。若服务器下发 `option 51 = 86400`，则 T1=43200 s、T2=75600 s。

**INIT-BACKOFF 重传模型**（RFC 2131 §4，弥补 UDP 不可靠）：

- 发送 DISCOVER/REQUEST 后启动 `INIT-RETRANSMIT`（默认 4 s，允许 0–1000 ms 随机抖动，防同时风暴）；无应答则指数退避重传，直到选定时间点后改为每 60 s 一次，无限重试。
- 客户端**必须忽略** `xid` 与当前会话不匹配的报文；BOUND 状态收到发给旧地址的广播也要丢弃。

---

## 📦 3. 报文格式逐字段表

头部**固定 240 字节**（含 4 B magic cookie），字段与 BOOTP 完全对齐：

| 偏移 | 字段 | 长度 | 方向 | 说明 |
| :--: | :--- | :--: | :--: | :--- |
| 0 | **op** | 1 B | 双向 | 1 = BOOTREQUEST（客户端→服务器），2 = BOOTREPLY（服务器→客户端）。DHCP 复用此字段，不另设请求/响应位 |
| 1 | **htype** | 1 B | 双向 | 硬件地址类型（沿用 ARP hardware type 号段）：`1` = Ethernet，`6` = IEEE 802，`16` = InfiniBand；绝大多数场景恒为 1 |
| 2 | **hlen** | 1 B | 双向 | 硬件地址长度，Ethernet = **6** |
| 3 | **hops** | 1 B | 双向 | 经过的 DHCP 中继数。客户端置 0，中继转发前 +1；超上限（常配 16）丢弃以**防环** |
| 4 | **xid** | 4 B | 双向 | 事务 ID（transaction ID）：客户端随机数，一轮 DORA 内不变，服务器原样回填；两侧用来匹配会话与丢弃杂包。（部分中文资料把偏移 4–7 写作 "xsec"，实为 `xid`；`secs` 在偏移 8–9） |
| 8 | **secs** | 2 B | C→S | 从本次地址获取开始计秒（BOOTP 时代留给 TFTP 引导就绪的时间）。DHCP 中继/服务器可用于判断等待时长与优先级 |
| 10 | **flags** | 2 B | 双向 | bit0 = **B（Broadcast）**：1 = 服务器必须以广播回包，0 = 客户端可收单播。bit1 = MAC 位（RFC 6494，标识 chaddr 来源），bit2–15 保留置 0 |
| 12 | **ciaddr** | 4 B | 双向 | Client IP address：客户端**已在使用**的地址。DORA 前三包恒 `0.0.0.0`；RENEW/REBIND/INFORM/DECLINE/RELEASE 填自身地址 |
| 16 | **yiaddr** | 4 B | S→C | "Your" IP address：服务器分配给客户端的地址（OFFER 中为提议，ACK 中为正式租约地址） |
| 20 | **siaddr** | 4 B | S→C | Server IP address：下一跳引导服务器（TFTP/Boot Server）地址。纯配置场景常为 0 或与 opt 54 同值；**PXE/无盘启动必须正确** |
| 24 | **giaddr** | 4 B | 双向 | Relay Agent IP address：中继**面向客户端一侧接口**的 IP。客户端直连场景恒为 0；它同时决定服务器选**哪个作用域/池**（见 §6） |
| 28 | **chaddr** | 16 B | 双向 | Client Hardware Address：前 `hlen` 字节为 MAC，其余补 0。是"客户端标识"的兜底（与 opt 61 并存时优先 opt 61） |
| 44 | **sname** | 64 B | S→C | Server host name（NUL 结尾字符串，可选）；现代实现多留空，服务器身份由 opt 54 表达 |
| 108 | **file** | 128 B | S→C | Boot file name：引导文件路径（`pxelinux.0`、`undionly.kpxe` 等），可被 opt 67 覆盖 |
| 236 | **options** | 可变 | 双向 | 必须以 **magic cookie** 开头：4 字节 `0x63 0x82 0x53 0x63`（点分十进制 **99.130.83.99**，十六进制字节串常写作 `63.82.53.63`），随后 TLV 选项，以 `255`（End）收尾 |

> [!NOTE] 线上帧长估算
> 240 B DHCP（含 cookie、不含任何业务选项） + 8 B UDP + 20 B IPv4 + 14 B Ethernet = **282 B**（不含 FCS 与前导码）。带全量选项的典型包约 300–340 B。

---

## 🧩 4. Options 重点表

### 4.1 选项通用编码

```text
 8 bit        8 bit         8 × length bit
┌────────┬──────────┬──────────────────────────┐
│ Code   │ Length L │        Data              │
└────────┴──────────┴──────────────────────────┘
```

- 每个选项：**1 B 类型 + 1 B 长度 + L B 数据**；一个报文内同一 Code **不得出现两次**（53/54 等强制唯一）。
- `0` = Pad（填充对齐），`255` = End（必须在末尾）。
- 客户端用 **opt 55 Parameter Request List** 点名需要的选项；服务器可回复未点名的项（除非配置了严格匹配）。

### 4.2 Option 53 — DHCP Message Type（必查项）

| 码值 | 名称 | 方向 | 出现位置 |
| :--: | :--- | :--: | :--- |
| **1** | DHCPDISCOVER | C → S | 找服务器 |
| **2** | DHCPOFFER | S → C | 应答 Discover |
| **3** | DHCPREQUEST | C → S | 选路确认 / RENEW / REBIND / INIT-REBOOT（四种语义靠 ciaddr 与 opt 50/54 区分） |
| **4** | DHCPDECLINE | C → S | 通告地址冲突（ARP 探测发现已被占） |
| **5** | DHCPACK | S → C | 确认租约（或 INFORM 的参数应答） |
| **6** | DHCPNAK | S → C | 拒绝（地址不属于该网段/已被占用/已过期） |
| **7** | DHCPRELEASE | C → S | 主动放弃地址 |
| **8** | DHCPINFORM | C → S | 已有地址，仅索取参数 |
| 9 | DHCPBONDING（ZONE） | — | RFC 2489（BoSON），实验性，几乎不遇 |
| 10 | DHCPLEASEQUERY | C → S | RFC 4388，运维查询（需授权） |
| 11 | DHCPLEASEUNASSIGNED | S → C | Leasequery 应答：地址未分配 |
| 12 | DHCPLEARN | C → S | RFC 4388，邻居学习探测 |
| 13/14 | DHCPV4QUERY / DHCPV4-RESPONSE | — | RFC 8723/7598（DHCPv4-over-DHCPv6） |

### 4.3 高频选项速查

| Code | 名称 | 长度/格式 | 作用与排错要点 |
| :--: | :--- | :--- | :--- |
| **53** | DHCP Message Type | 1 B | 判读包子的第一步；见 §4.2 |
| **54** | Server Identifier | 4 B IP | 服务器的 IP 身份。**REQUEST 必须回填**，多宿主服务器/多作用域时决定"跟哪台成交"；NAK 也靠它溯源 |
| **50** | Requested IP Address | 4 B IP | 客户端"我想要这个地址"：选路阶段的 REQUEST 与 INIT-REBOOT 携带；RENEW 时**不携带** |
| **51** | IP Address Lease Time | 4 B，秒 | 租期；`0xffffffff` = 无限。地址池吞吐与安全（防地址被长期占用）的平衡值：办公 8 h–1 天，公共 Wi-Fi 1–2 h |
| **58** | Renewal Time (T1) | 4 B，秒 | 到达即单播 RENEW；缺省按 0.5 × lease |
| **59** | Rebinding Time (T2) | 4 B，秒 | 到达即广播 REBIND；缺省按 0.875 × lease |
| **1** | Subnet Mask | 4 B IP | 下发掩码；与服务器作用域掩码一致，否则会出现"能上内网不能上外网"（掩码错→网关不在同一子网） |
| **3** | Router（默认网关） | 4 B × N IP | 按优先级排序的网关列表；客户端取第一个作默认路由（Windows DHCP 控制台可给每条网关配 metric） |
| **6** | Domain Name Servers | 4 B × N IP | DNS 服务器；与 opt 15（Domain）、opt 119（Domain Search List）配合 |
| **15 / 119** | Domain Name / Domain Search | 变长 | 后缀补全与搜索列表（119 是压缩标签编码，RFC 3396） |
| **26 / 28** | Interface MTU / Broadcast Address | 2 B / 4 B | 特殊链路（隧道、PPPoE-over、组播）会用到 |
| **42 / 4** | NTP Servers / Time Offset | 变长 | 时钟同步；VoIP/机顶盒常见 |
| **43** | Vendor Specific Information | 变长（嵌套 TLV） | 厂商私有信息封装：**PXE 用它下发"服务类型 + 启动服务器列表"**（sub-option 43.5 = 服务类型，43.6 = boot servers）。AP 发现 AC 地址（option 43.241 / sub 0xF1）也是它 |
| **60** | Vendor Class Identifier | 变长 ASCII | 厂商类别串（PXE 客户端填 `PXEClient`），**服务器按它做策略分类**（vendor class / 用户类别） |
| **61** | Client Identifier | 变长（htype+hlen+id） | 客户端稳定标识。无线/多 VLAN 或需要"同一人同一地址"时用，优先级高于 chaddr；注意 MAC 随机化会改变它 |
| **12 / 81** | Host Name / FQDN | 变长 | 客户端上报主机名；81 用于服务器反向写 DNS（配合 opt 252? 无关，见 RFC 4701） |
| **55** | Parameter Request List | 变长 Code 列表 | 客户端点名要哪些选项 |
| **57** | Maximum DHCP Message Size | 2 B | 客户端能收的最大 UDP 载荷；小于 576 时服务器必须做选项裁剪 |
| **66 / 67** | TFTP Server Name / Bootfile Name | 变长 | 无盘/PXE、零信任 AP 自动配置常用 |
| **82** | Relay Agent Information | 变长（子选项） | RFC 3046：中继写入 `circuit-id`/`remote-id`，运营商城域网据此做用户识别与地址策略 |
| **249 / 121** | Classless Static Route | 多 CIDR 条目 | 一次性下发"到指定网段走指定网关"（VPN/分域访问场景）；121 是 RFC 3442，249 是微软早期专用码 |
| **0 / 255** | Pad / End | — | 填充与结束标记，缺 255 的包多为畸形/截断 |

> [!TIP] 一句话 PXE
> PXE = "DHCP + BOOTP 引导"：客户端 opt 60 = `PXEClient`，DHCP 用 **siaddr + file** 或 **option 43（sub-option 43.5/43.6）** 指路，客户端再用 **DHCPREQUEST（opt 53=3，opt 55 含 43）** 向"代理 DHCP/PXE 服务"索取启动镜像，随后 TFTP 下载 NBP。

---

## 🗂️ 5. 地址池、绑定与选项继承

### 5.1 服务端概念

| 概念 | 说明 | 典型配置（ISC / Kea / Windows） |
| :--- | :--- | :--- |
| **作用域（Scope / subnet）** | 一个 IP 网段 + 掩码 + 该段参数 | `subnet 10.0.10.0 netmask 255.255.255.0 { … }` / Windows "作用域 10.0.10.0" |
| **地址池（Pool / range）** | 可动态分配的连续区间 | `range 10.0.10.100 10.0.10.199;` |
| **排除/保留** | 静态网关、打印机、服务器不入库 | `host prn01 { hardware ethernet …; fixed-address …; }` / Windows "排除范围" + "保留" |
| **绑定（Binding / Lease）** | (客户端标识, 地址, 起止时间, 主机名, 已下发选项) 的记录 | ISC：`/var/lib/dhcp/dhcpd.leases`；Kea：MySQL/PostgreSQL 或 memfile；Windows：Jet DB `dhcp.mdb` |
| **冲突标记** | DECLINE 或服务器 ping 探测判定的坏地址，冻结一段时间 | ISC `ping-check true;`（先 ICMP echo + 超时 1 s）；冲突记录在 lease 文件 |
| **租期回收** | 过期租约可被重新分配；RELEASE 立即回池 | 过期 ≠ 立即回收（Kea/ISC 有 `expire-lag`/`default-lease-time` 语义差异） |
| **双机热备** | 避免单点 | Windows：拆分作用域 80/20（80/20 split scope）；ISC/Kea：RFC 7283 Failover **M/N** 模式（load-balance 或 hot-standby，含 `mclt` 与 pool partner down 时间） |

### 5.2 选项继承顺序（后者优先覆盖前者）

```text
服务器全局（global / server）
   → 作用域 / 网段（subnet / scope）
      → 地址池（pool）
         → 保留 / 主机（reservation / host / class）
            → 中继侧或策略插入（option 82 派生的子网选择）
```

> [!WARNING] 池耗尽的表现
> 池满时**部分服务器静默不回包**（ISC 记日志 `No free leases`；Kea `DHCPNOACK`），客户端表现就是"拿不到 IP、反复 DISCOVER 超时后落到 169.254.x.x"。判据：服务器统计 `InUse / MaxConfigurableAddresses`，或抓包看到 Discover 有来无回但中继正常转发。

---

## 📮 6. DHCP Relay 中继与 GIADDR

三层设备（路由器 / 交换机 SVI / Windows「DHCP 中继代理」角色 / Linux `isc-dhcp-relay`、`kea-dhcp4` relay mode）跨网段转发：

```text
VLAN 10 客户端                SVI 10.0.10.1 (=giaddr)             服务器 172.16.0.5
0.0.0.0:68 → 255.255.255.255:67 ──► 改写：giaddr=10.0.10.1, hops+1, 源 IP=10.0.10.1
                                    ──────── 单播 10.0.10.1:67 → 172.16.0.5:67 ────────►
                                    ◄─────── 目的 IP = giaddr:67（服务器一定单播给中继）
 ◄── 中继向 VLAN 10 广播（或按 chaddr 定向转发），目的端口 68
```

**GIADDR 的三个作用（必须记牢）**：

1. **选池**：服务器用 `giaddr` 匹配自己声明的 `subnet`，决定从哪个作用域/池分配地址——这是"跨网段拿到地址"的唯一依据。客户端真实子网对服务器**不可见**。
2. **回程定位**：服务器一律把应答**单播回 `giaddr:67`**，忽略 B 位；中继再据 `chaddr` / opt 61 在下发的绑定表里找到客户端所在 VLAN/端口并转发。
3. **防环与限跳**：`hops` 递增，超限丢弃；giaddr 非 0 的包在直连端口出现往往是攻击或错误配置信号（snooping 会丢弃）。

常见故障对照：

| 现象 | 根因方向 |
| :--- | :--- |
| 中继配了但客户端拿不到地址，`giaddr=0` 出现在服务器日志 | 客户端到中继的二层不通（VLAN/access 口配错、native VLAN、端口未放行） |
| 服务器日志 `DHCPDISCOVER from xx via 10.0.10.1` 但 `No subnet declaration for 10.0.10.0` | 缺该 giaddr 对应网段的 `subnet` 声明（掩码写错也会这样） |
| 拿到了**错误网段**的地址 | giaddr 与作用域映射错配 / 双中继重复插入 / option 82 策略错误 |
| 只对某几台 VLAN 失效 | ACL 只放行了部分中继 IP；或 UDF/STP 期间丢广播 |

---

## 🧭 7. 多网卡与 metric 选路

- 客户端为**每个接口独立**运行一套 DHCP 状态机，各自获得地址与默认路由；DHCP 本身**不做选路**，最终出接口由路由表决定。
- Windows 默认启用 **Automatic Metric**：按链路速度给接口路由打 metric，**值越小越优先**；速度相同则**路由条目少的接口优先**，再相同按**适配器 GUID 顺序**。

Windows 10 及以后自动度量（接口级基线，实际值还叠加路由条数）：

| 链路速度 | 有线等其它介质 | 无线（802.11 / Native802_11） |
| :--- | :--: | :--: |
| ≥ 100 Gb | 5 | — |
| ≥ 10 Gb | 15 | — |
| ≥ 2 Gb | 20 | 25 |
| ≥ 500 Mb | —（1 Gb 落在 20） | 30 |
| ≥ 200 Mb | 25 | 35 |
| ≥ 150 Mb | — | 40 |
| ≥ 80 Mb | — | 45 |
| ≥ 20 Mb | 45 | 55 |
| ≥ 4 Mb | 55 | 65 |
| ≤ 500 Kb | 75 | 85 |

- 半双工介质（早期无线）按**有效速率折半**参与计算。
- 查看与手工固定：

```powershell
Get-NetIPInterface | Sort-Object AddressFamily,InterfaceMetric | Format-Table InterfaceAlias,AddressFamily,InterfaceMetric,Dhcp
Set-NetIPInterface -InterfaceAlias "Ethernet" -InterfaceMetric 10          # 关自动度量并固定
route print 0.0.0.0                                                        # 看默认路由跃点数
ipconfig /all
```

> [!NOTE] 排错经验
> "双网卡只有一张能上外网 / 上网时断时续"，九成是**两个接口都下发了默认网关（opt 3）** 导致路由回跃点抖动，或**非对称路由**（走 A 出、走 B 回）。企业做法：只允许一个接口有默认网关，内网段用 option 121/249（Classless Static Route）定向下发。

---

## ⚠️ 8. APIPA（169.254/16）：拿不到地址时的行为

| 项目 | 内容 |
| :--- | :--- |
| 标准 | 链路本地编址 RFC 3927（IPv4 Link-Local），冲突检测 RFC 5227（ARP Probe/Announce） |
| 地址段 | `169.254.0.0/16`（`169.254.0.0 – 169.254.255.255`），**不可路由**，仅同链路可达 |
| 命名 | Windows 称 **APIPA**（Automatic Private IP Addressing）；macOS/iOS 称 self-assigned / IPv4LL；Linux 需 `systemd-networkd` 的 `IPvLinkLocalAddressing` 或 `avahi-autoipd` 才有 |
| 选址过程 | 随机挑地址 → **ARP Probe**（RFC 5227：3 次，间隔 2 s）无回应则占用 → 发免费 ARP 通告 → 被质疑（有人回 ARP）则重新挑选，最多 5 次防御 |
| 后续重试 | Windows 会周期性重试 DHCP（默认约每 4–5 分钟一次）；`ipconfig /renew` 可强制 |
| 典型判读 | `ipconfig` 看到 `169.254.x.x` + 掩码 `255.255.0.0` + **无默认网关** = "DISCOVER 有去无回"，问题在链路/服务器/池/中继，而不是 DNS 或网关 |

> [!CAUTION] 别把 APIPA 当"能用"
> 只有单栈 IPv4 主机才会暴露这个问题：IPv6 永远自带 `fe80::/10` 链路本地地址，所以"看起来能 ping 通邻居但打不开网页"往往说明 **DHCPv6-PD / DNS 下发**失败，而不是完全没配网。

---

## 🛡️ 9. 安全：DHCP 欺骗、Snooping 与耗尽攻击

### 9.1 威胁模型

| 攻击 | 手法 | 后果 |
| :--- | :--- | :--- |
| **Rogue DHCP Server（服务器欺骗）** | 私接设备/虚拟机跑 `dhcpd`/`dnsmasq --dhcp-range`，回 Offer/ACK | 客户端拿到假网关/DNS → **中间人、DNS 劫持、全流量导流** |
| **DHCP 耗尽（Starvation）** | 伪造海量随机 `chaddr`/opt 61 持续 DISCOVER，占满地址池 | 真实用户拿不到地址（DoS）；常作 **STP 篡改 / DHCP 快照** 的前置 |
| **参数篡改 / 重放** | 中间人改 ACK 里的 opt 3/6，或重放旧 ACK（短租期、NAK 洪水） | 断网、被引导到攻击者 DNS、频繁重新 DORA |
| **中继侧伪造** | 攻击者包插入伪造 option 82 或伪装成 giaddr 非 0 的包 | 逃计费/审计、错误网段分配 |

### 9.2 DHCP Snooping（交换机侧，事实标准）

核心是把端口分成 **trusted / untrusted**：

| 机制 | 说明 |
| :--- | :--- |
| **信任端口** | 只允许上联/中继/服务器方向的端口（如 `interface Te1/0/1` → `ip dhcp snooping trust`）收到**来自服务器**的 Offer/ACK；非信任端口收到就是 Rogue → 丢弃（可 err-disable） |
| **不信任端口限制** | 只放行客户端→服务器方向（DISCOVER/REQUEST/DECLINE/RELEASE，端口 67 为目的），并可限速 |
| **绑定表（binding table）** | 监听 ACK 记录 `IP – MAC – VLAN – 端口 – 租期 – 类型`，是后续 **IP Source Guard / DAInspect / Dynamic ARP Inspection** 的数据源 |
| **速率限制** | `ip dhcp snooping limit rate <n>`（每端口每秒可收的 DHCP 报文数，超出即 shutdown/告警），抵御耗尽洪水 |
| **选项校验** | `ip dhcp snooping information option allow-untrusted` / 校验、重写 option 82（`format circuit-id`），防伪造 |
| **数据库持久化** | `ip dhcp snooping database write-time <min>`，重启不丢绑定 |

参考配置骨架（Cisco IOS，思科官方 *Configuring DHCP Snooping* 指南）：

```text
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
interface GigabitEthernet0/1          ! 上联/中继口 = 信任
 ip dhcp snooping trust
 ip dhcp snooping limit rate 15
interface GigabitEthernet0/2          ! 接入口 = 非信任（默认）
 ip dhcp snooping limit rate 5
ip dhcp snooping database flash:/dhcp-snooping.db
```

> [!TIP] 配套加固
> 端口安全（`switchport port-security maximum 2`）、STP Root Guard / BPDU Guard、控制平面策略（CoPP）、以及服务器侧 **802.1X / MAC 认证**；仅 Snooping 不能防"合法端口的少量伪造标识"，需要绑定表 + 租期审计结合。

---

## 🔧 10. 排错清单（命令 + 抓包判读）

### 10.1 客户端命令

```bash
# Windows
ipconfig /all                 # 看 DHCP Enabled / 是否 169.254 / 拿到哪个服务器
ipconfig /release "Ethernet0" # 放弃租约（发 RELEASE）
ipconfig /renew   "Ethernet0" # 重新 DORA（有缓存地址时先尝试 INIT-REBOOT）
ipconfig /showclassid Ethernet0 ; ipconfig /setclassid Ethernet0 <id>   # 厂商类别（策略分类）
getmac /v ; route print ; arp -a
Get-NetIPInterface | ft InterfaceAlias,InterfaceMetric,Dhcp   # 多网卡选路
Get-WinEvent -LogName "Microsoft-Windows-Dhcp-Client/Admin"  # 客户端失败事件
netsh int ipv4 show dynamicport tcp ; netsh int ip reset     # 栈重置（最后手段）

# Linux
ip -4 addr show ; systemctl restart systemd-networkd | NetworkManager
nmcli -f IP4.ADDRESS,IP4.GATEWAY,IP4.DNS con show "Wired 1"
journalctl -u systemd-networkd -b | grep -i dhcp
dhclient -v eth0 ; dhcpcd -d eth0        # 手工前台跑，看逐包时序
```

### 10.2 服务器侧

```bash
# ISC DHCP / Kea
dhcpd -t -cf /etc/dhcp/dhcpd.conf            # 语法校验（改完必做）
grep -c "^lease" /var/lib/dhcp/dhcpd.leases  # 当前绑定数
tail -f /var/log/syslog | grep -Ei 'dhcp|No free leases'
kea-shell --service dhcp4 lease4-get-all     # Kea：查租约/统计
# Windows DHCP Server
Get-DhcpServerv4Scope | ft Name,StartRange,EndRange
Get-DhcpServerv4Statistics -ScopeId 10.0.10.0 |
  ft MaxConfigurableAddresses,InUse,FreeWaiting,PendingFreed   # 池耗尽一眼可判
Get-DhcpServerv4Lease -ScopeId 10.0.10.0 -Address 10.0.10.123
Test-DhcpServerv4Failover -ComputerName dhcp01
```

### 10.3 抓包过滤与判读

| Wireshark 过滤器 | 用途 |
| :--- | :--- |
| `udp.port == 67 || udp.port == 68` | 只看 DHCP（最通用；含中继 67↔67） |
| `bootp` / `dhcp` | BOOTP/DHCP 全量 |
| `bootp.option.dhcp == 1`（2/3/5/6） | 只看某类消息（Discover/Offer/Request/ACK/NAK） |
| `bootp.flags & 0x8000 == 0` | 检查广播位（Wireshark 显示位序按 MSB 起，实测用 `bootp.broadcast_code`/列 `Flags`） |
| `bootp.giaddr != 0` | 只有经过中继的包 |
| `bootp.xid == 0x12345678` | 串起一轮完整 DORA |
| `bootp.chaddr == aa:bb:cc:dd:ee:ff` | 追单个终端 |
| `bootp.option.subcode == 50` / `== 54` | 请求地址 / 服务器标识 |

判读要诀：

- **只有 DISCOVER、无 OFFER** → 二层不通 / 中继未配或 giaddr 网段无声明 / 服务器停 or 池耗尽 / ACL 拦 67。
- **有 OFFER 无 ACK** → 客户端 REQUEST 未被服务器接受（opt 50/54 不匹配、NAK 被丢弃）、或客户端只监听广播而服务器回单播。
- **收到 NAK（opt 53=6）** → 终端被挪了网段 / 缓存地址过期 / 保留冲突，客户端会退回完整 DORA（正常自愈，若反复 NAK 查作用域与掩码）。
- **同一 xid 多次 DISCOVER，间隔≈4 s、8 s、16 s** → 典型的 INIT-BACKOFF 指数退避，说明服务器侧没响应。
- **`secs` 很大（>60）反复重传** → 客户端在等；配合 `hops=0` 判断"根本没到中继发"。
- **抓包点在服务器却看不到客户端广播** → 广播不过路由器，属正常，应在**客户端 VLAN 内**或中继出口镜像抓。
- **多个 OFFER 属正常**；只要最终有一个 ACK，且 xid 一致即可。

---

## 🖼️ DHCP 报文与 DORA 时序图

![[DHCP.excalidraw]]

---

## 🔗 思考与延伸索引

- [[UDP]] — DHCP 的承载协议：无连接、无重传，因此超时/退避/`xid` 匹配全在应用层实现
- [[IP]] — `0.0.0.0`、`255.255.255.255` 有限广播、`169.254/16` 链路本地与子网掩码的意义
- [[TCP]] — 对照：DHCP 完成后上层连接才登场；DHCP 丢包不会重传，TCP 会，理解两者"可靠性责任"的差异
- [[IP路由与metric]] — 多接口/多默认网关时的选路细节（若已建该笔记）

---

## 📚 参考资料

1. **IETF RFC 2131** — *Dynamic Host Configuration Protocol*, Droms (Ed.), 1997（STD 53；报文格式、状态机、T1/T2、中继行为）.
2. **IETF RFC 2132** — *DHCP Options and BOOTP Vendor Extensions*（选项编码、53/54/50/51/58/59、cookie、opt 43/60）.
3. **IETF RFC 951 / RFC 1542** — *Bootstrap Protocol (BOOTP)*（`siaddr`、`file`、`htype/hlen/hops` 的原始语义）.
4. **IETF RFC 3046** — *The DHCP Relay Agent Information Option*（Option 82 circuit-id / remote-id）.
5. **IETF RFC 3927** — *Dynamic Configuration of IPv4 Link-Local Addresses*（169.254/16 APIPA）.
6. **IETF RFC 5227** — *IPv4 Duplicate Address Detection*（ARP Probe / Announce / 防御流程）.
7. **IETF RFC 6842** — *Clarifications for the DHCP*（首个 DISCOVER 必须用源端口 68）.
8. **IETF RFC 7283** — *DHCPv4 Failover*（M/N、load-balance、`mclt`）.
9. **IETF RFC 4388** — *DHCP Leasequery*（消息类型 10/11/12 与运维查询）.
10. **Cisco** — *DHCP Snooping* 章节, *Cisco IOS Security Configuration Guide*（trusted/untrusted、binding table、`limit rate`、与 DAI/IPSG 联动）.
11. **Microsoft Learn** — *The Automatic Metric feature for IPv4 routes*（KB 299540，链路速度 → 度量值表）；*DHCP Server role documentation / Windows DHCP Options*（含 vendor 类别与 003 Router 度量）.
12. **ISC** — *DHCP 4.4 Administrator Reference Manual*（`ping-check`、`conflict`、failover、`dhcpd.leases`）；**Kea DHCP** 手册（lease DB、statistics、relay 模式）.
13. **IETF RFC 3396 / RFC 3442 (opt 121) / RFC 4701 (opt 81)** — 域名搜索列表编码、Classless Static Route、FQDN 选项.
