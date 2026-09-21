---
title: USB 协议完整技术手册
tags:
  - Embedded
  - Protocol
  - USB
  - Enumeration
  - Signal
  - Descriptors
status: Completed
created: 2026-09-21
---

# 🔌 USB 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：主机主导轮询（host-polled）、令牌驱动（token-based）的串行总线。设备**永不主动发起总线传输**，一切由 Host 的 token 包排期；物理层与协议层解耦，同一套包格式承载 LS/FS/HS 三档速率。
> - **内容范围**：体系与物理接线 · 上拉速率判定 · NRZI + 位填充 · 包格式与 PID/CRC · 事务与四种传输类型 · 枚举流程 · 描述符体系 · 帧/微帧调度 · USB 2.0/3.x 差异 · 常见坑
> - **推荐路径**：§1 物理层 → §2 信号编码 → §3 包格式（核心）→ §4 事务/传输类型 → §5 枚举 → §6 描述符 → §7 调度与差异
> - **一句话记忆**：**包（Packet）→ 事务（Transaction）→ 传输（Transfer）**：包是比特流单位，事务是 token+data+handshake 的一次握手，传输是应用视角的一段完整数据（控制/批量/中断/等时）。

---

## 🧩 1. 协议体系与物理层

### 1.1 三种角色与星型拓扑

| 角色 | 英文 | 职责 | 典型实现 |
| :--- | :--- | :--- | :--- |
| **主机** | Host / Root Hub + HCD | 唯一总线master：产生 SOF、发 token、分配带宽、驱动枚举 | PC xHCI 控制器、SoC USB OTG 控制器 |
| **集线器** | Hub | 下游端口扩展 + 连接状态检测 + 复位/挂起传播；HS Hub 内含 **TT（Transaction Translator）** 代 FS/LS 设备做高速事务 | USB 2.0 Hub、HSIC/PHY |
| **设备** | Device / Function | 被动响应寻址到自己的 token；上报端点/描述符；可兼 Hub | MCU（FS/LS device）、外置 PHY（ULPI/UTMI） |
| **双角色** | OTG / DRD | Host↔Device 角色协商（HNP）、会话请求（SRP） | 嵌入式 OTG 口 |

拓扑为**严格的分层星型（tiered star）**：主机端口 → Hub → 设备，最多 **5 层 Hub**（7 层级），**不能挂总线型共享介质**，任何一段点对点链路只服务一个下行设备。

```text
                     ┌────────────┐
                     │  Host (HCD)│  ← 唯一调度者，产生 SOF/token
                     └──────┬─────┘
                  ┌─────────┴─────────┐        ← 星型：每个端口一条点对点链路
             ┌────┴────┐         ┌────┴────┐
             │  Hub    │         │ Device  │
             └──┬───┬──┘         └─────────┘
          ┌─────┘   └─────┐
     ┌────┴────┐     ┌────┴────┐
     │ Device  │     │  Hub …  │   ← 最多 5 级 Hub 级联
     └─────────┘     └─────────┘
```

### 1.2 物理接线（USB 2.0 Type-A/B 4 线）

| 引脚 | 信号线 | 颜色（典型） | 作用 | 电平/电流 |
| :--: | :--- | :--- | :--- | :--- |
| 1 | **VBUS** | 红 | 主机供电 | $+5\text{ V} \pm 5\%$；单位负载 LS/FS $100\text{ mA}$，HS $500\text{ mA}$ |
| 2 | **D−** | 白 | 差分数据（负） | 差分 $\pm 200\text{ mV}$ 门限（LS/FS 单端 $0\sim 3.3\text{ V}$） |
| 3 | **D+** | 绿 | 差分数据（正） | 同上；FS/HS 设备在此挂上拉 |
| 4 | **GND** | 黑 | 信号地，**双端必须共地** | — |
| 壳 | Shield | — | 屏蔽层，抗共模/EMI | — |

> [!WARNING] 与 UART 的根本差异
> UART 只需 TX/RX 交叉 + GND 即可通；USB 必须有 **VBUS 供电回路 + 差分对 + 主机端接**。差分对 D+/D− **接反**会导致上拉落在错误线上，主机识别为「未知/低速」或报 `Cannot enable port`，是硬件首查项。

### 1.3 上拉电阻与速率判定（必背）

主机侧 D+、D− 各挂 **$15\text{ k}\Omega$ 下拉**（规范 $14.25\sim 23.5\text{ k}\Omega$）到地，空闲时为 SE0；设备侧用一个 $1.5\text{ k}\Omega$ 上拉（$90\%\sim 110\%$）到 $3.0\sim 3.6\text{ V}$ 表示「我已插入并声明速率」。

$$
V_{\text{IDLE}} = 3.3\text{ V} \times \frac{15\text{ k}\Omega}{15\text{ k}\Omega + 1.5\text{ k}\Omega} \approx 3.0\text{ V},\qquad
I_{\text{pull-up}} = \frac{3.3\text{ V}}{16.5\text{ k}\Omega} \approx 0.2\text{ mA}
$$

| 上拉位置 | 空闲差分状态 | 主机判定（`dp_status`/`dm_status`） | 速率档 |
| :--- | :--- | :---: | :--- |
| **D+ → 3.3 V** | J 态（D+ 高、D− 低） | `dp=1, dm=0` | **全速 Full-Speed 12 Mbps**（HS 设备先以 FS 枚举，再由复位握手升速） |
| **D− → 3.3 V** | 反转 J 态（D− 高、D+ 低） | `dp=0, dm=1` | **低速 Low-Speed 1.5 Mbps**（键鼠等 HID 常用） |
| **D+ 和 D− 都下拉** | SE0（单端零） | `dp=0, dm=0` | **未连接 / 已断开**（或复位/挂起期间） |
| 无上拉、有 VBUS | SE0 | 端口不变化 | 不枚举（"假插"，多为线缆/上拉漏焊） |

- **高速（480 Mbps）不用上拉区分**：设备仍以 FS 上拉枚举，主机在**复位结束后发 Chirp K**，设备回一串 **Chirp K-J** 序列声明 HS 能力；双方随后启用 **24 MHz 时钟 + HS 电流模式驱动**（差分摆幅约 $400\text{ mV}$）与 $45\,\Omega$ 端接。
- USB 3.x 在此之上另加 **SSTX±/SSRX± 两对差分**（SuperSpeed 5/10 Gbps），并保留 D+/D− 兼容 2.0 枚举。

---

## 📡 2. 信号层：编码、位填充与速率

### 2.1 NRZI 编码（Non-Return-to-Zero Inverted）

USB 数据流用 **NRZI**：**逻辑 1 = 不跳变，逻辑 0 = 在位边界翻转电平**。

| 数据位 | NRZI 行为 | 说明 |
| :--: | :--- | :--- |
| **1** | 保持上一电平 | 连续 1 会长期无跳变 → 接收方失去同步 |
| **0** | 电平翻转（J↔K） | 提供边沿，但连续 0 同样造成固定图案 |

```text
数据位      : 1    1    0    0    1    0    1    1
NRZI 波形   : ─────┐    ┌────┐    ┌────┐    ─────
(J/K)         J    └─ K ─┘ J    └─ K ─┘ J   └ K    J   └── 1 不变，0 翻转
```

> [!NOTE] 为何还要位填充
> NRZI 自身不含时钟信息，USB 也无独立时钟线（对照 [[UART]] 靠波特率自同步、[[SPI]] 靠 SCK）。**「NRZI + 位填充」共同构成自时钟编码**：靠不断强制出现的边沿让接收方 PLL 保持锁定。

### 2.2 位填充（Bit Stuffing）

- 发送端：从 **SYNC 之后到 CRC 末位**（不含 EOP）之间，**每出现连续 6 个 1 就插入一个 0**，强制制造一次 NRZI 翻转。
- 接收端：检测到连续 6 个 1 后丢弃下一位还原原始流；若检测到 **7 个连续 1**，判定位填充错误（Bit-Stuff Error），整包丢弃并计入 CRC/帧错误。
- 开销：最坏每 6 位数据补 1 位 →

$$
\eta_{\text{stuff,max}} = \frac{6}{7} \approx 85.7\% \quad\Rightarrow\quad \text{额外开销} \le 16.7\%
$$

### 2.3 线路状态：SYNC / J / K / SE0 / EOP

| 状态 | D+ / D− | 含义与用法 |
| :--- | :---: | :--- |
| **J 态** | FS: H / L；LS: L / H | 空闲（Idle）状态；包尾恢复位 |
| **K 态** | J 的反相 | 包的起始（SE 后）、NRZI 数据电平之一 |
| **SE0** | 双低（$<0.3\text{ V}$） | 复位（**持续 $\ge 10\text{ ms}$**）、错误、挂起；LS 设备用它识别 reset |
| **SE1** | 双高 | **非法**，视为故障 |
| **SYNC** | `KJKJKJKK` | 8 bit 前导（NRZI 编码的 `00000001`），让接收方对齐位边界，**不参与 CRC** |
| **EOP** | SE0(2 bit) + J(1 bit) | 包结束标志；HS 下即 $3 \times 2.083\text{ ns} \approx 6.25\text{ ns}$ |
| **Chirp K/J** | HS 专用 | 复位期间的 HS 能力握手与分事务（split）标记 |

### 2.4 速率档与位时间

| 规范 | 档位 | 标称位率 | 单位间隔 UI | 每帧位数 | 编码 | 有效吞吐（典型） |
| :--- | :--- | :---: | :---: | :---: | :--- | :---: |
| USB 1.0/1.1 | **Low-Speed** | $1.5\text{ Mbps}$ | $666.7\text{ ns}$ | 1500 | NRZI+位填充 | $\approx 0.1\text{ MB/s}$ |
| USB 1.1 | **Full-Speed** | $12\text{ Mbps}$ | $83.33\text{ ns}$ | 12000 | NRZI+位填充 | $\approx 1.1\text{ MB/s}$ |
| USB 2.0 | **High-Speed** | $480\text{ Mbps}$ | $2.083\text{ ns}$ | 60000/微帧 | NRZI+位填充（UTMI） | $\approx 35\sim 42\text{ MB/s}$ |
| USB 3.0/3.1 Gen1 | **SuperSpeed** | $5\text{ Gbps/lane}$ | $200\text{ ps}$ | — | **8b/10b** + 包头（HDP）+ SKP | $\approx 400\text{ MB/s}$ |
| USB 3.1/3.2 Gen2 | **SuperSpeed+** | $10\text{ Gbps/lane}$ | $100\text{ ps}$ | — | **128b/132b** | $\approx 850\text{ MB/s}$ |

- **时钟精度要求**：LS/FS 收发端振荡器容差 $\pm 0.25\%$（$2500\text{ ppm}$），HS 端 PHY 需 $\pm 500\text{ ppm}$ 且靠恢复时钟工作 —— 远严于 [[UART]] 的 $\le\pm 2\%$ 结论，因为 USB 一帧/一包内位数多、无停止位可重新对齐。

> [!TIP] 与 UART/SPI 的同步方式对照
> | 维度 | UART | SPI | USB 2.0 |
> | :--- | :--- | :--- | :--- |
> | 时钟 | 无（约定波特率） | 有 SCK | 无（NRZI + 位填充自同步） |
> | 成帧 | 起始/停止位 | 片选 CS 拉低 | SYNC 前导 + EOP(SE0) |
> | 寻址 | 无（点对点） | 片选线 | 7 bit 地址 + 4 bit 端点 |
> | 主控 | 双方对等 | 主从固定 | Host 轮询，Device 只应答 |

---

## 📦 3. 包格式（Packet Format，核心）

### 3.1 包的字段布局

```text
├─────────────────────────── 一个包（Packet） ───────────────────────────┤
┌─────────┬───────────────┬───────────┬───────────┬────────────┬─────────┬───────────┐
│  SYNC   │      PID      │   ADDR    │   ENDP    │    DATA    │  CRC    │    EOP    │
│ 8 bit   │ 4 + 4 bit     │  7 bit    │  4 bit    │ 0~1024 B   │ 5/16 bit│ SE0+J     │
│ 前导同步 │ 类型+反码校验 │  设备地址  │  端点号   │  有效载荷   │ 检错    │  3 bit    │
└─────────┴───────────────┴───────────┴───────────┴────────────┴─────────┴───────────┘
   不参与 CRC │◄──────────── CRC5 覆盖 PID+ADDR+ENDP ────────────►│
              │◄────────────── CRC16 覆盖 DATA（数据包） ─────────►│
```

| 字段 | 长度 | 存在性 | 说明 |
| :--- | :--: | :--- | :--- |
| **SYNC** | 8 bit | 所有包 | `KJKJKJKK`，位同步；最后 2 bit（SE2）标识 PID 起点 |
| **PID** | 8 bit | 所有包 | **4 bit 类型码 + 4 bit 按位反码**；接收方校验二者互补，不互补即 PID 损坏，整包丢弃且**不回 handshake** |
| **ADDR** | 7 bit | token 包 | 设备地址，$0\sim 127$；`0` 为枚举前的默认地址 → **最多 127 个设备** |
| **ENDP** | 4 bit | token 包 | 端点号 $0\sim 15$（EP0 为双向控制端点） |
| **DATA** | 0~1024 B | data 包 | 长度须为 8 bit 整数倍；FS ≤1023 B，HS 高带宽等时 1024 B；**0 字节 DATA 包合法**（用于状态阶段/短包终止） |
| **CRC** | 5 / 16 bit | token / data | 见 §3.3 |
| **EOP** | 3 bit | 所有包 | 2 bit SE0 + 1 bit J 恢复 |

> [!NOTE] 三类包的字段差异
> - **Token / SOF 包**：`SYNC + PID + (ADDR|ENDP 或 11 bit 帧号) + CRC5 + EOP`，固定 **30 bit** 时间（FS 一帧 12000 bit → 仅占 $0.25\%$）
> - **Data 包**：`SYNC + PID + DATA + CRC16 + EOP`
> - **Handshake 包**：`SYNC + PID + EOP`，**无 DATA、无 CRC** → 只有 8 bit PID 承载信息，故 PID 反码校验格外重要

### 3.2 PID 四大类（USB 2.0 规范 Table 8-6）

**① Token 类（令牌，Host → Device）**

| PID | 发起 | 载荷字段 | 作用 | 后续阶段 |
| :--- | :---: | :--- | :--- | :--- |
| **OUT** | Host | ADDR+ENDP+CRC5 | 主机要**向设备写**：本事务的 data 包是主机发出的 | Device 回 ACK / NAK / STALL |
| **IN** | Host | ADDR+ENDP+CRC5 | 主机要**向设备读** | Device 发 data 包 或 NAK / STALL |
| **SETUP** | Host | ADDR+ENDP+CRC5 | 控制传输建立阶段专用；**忽略数据翻转、必须无条件接受**，data 为 8 B 标准请求 | Device 回 ACK |
| **SOF** | Host | 11 bit 帧号 + CRC5 | 帧/微帧起始广播，不承担寻址、**无 handshake** | — |

**② Data 类（Data → 承载载荷）**

| PID | 作用 |
| :--- | :--- |
| **DATA0** | 偶数编号数据包（ toggle 起点，SETUP 数据固定为 DATA0） |
| **DATA1** | 奇数编号数据包，与 DATA0 交替实现**数据翻转同步** |
| **DATA2** | 等时传输中「第 2 个可接受但不期望」的包（高带宽等时/错误恢复用） |
| **MDATA** | 高带宽等时传输（HS）中的第三种翻转态 |

**③ Handshake 类（应答，无 CRC）**

| PID | 由谁发 | 语义 | 对上位机的可见结果 |
| :--- | :---: | :--- | :--- |
| **ACK** | 接收成功方 / 发送方已被收 | 已正确收下（或已发出且被收）；CRC 通过、位填充无误 | 事务成功 |
| **NAK** | Device | 「数据没准备好 / 缓冲区暂时不能收」，**流控**，事务失败但可重试 | 主机稍后再发同一 token |
| **STALL** | Device | 「这个端点被 Halt，重试也没用」，需 `CLEAR_FEATURE(ENDPOINT_HALT)` 恢复 | 端点错误 / 控制请求不支持 |
| **NYET** | Device（HS） | 包已收下但**下一包不可再发**：HS 批量/控制的 PING 协议、等时反馈 | 主机不再重试当前 data |

**④ Special 类（HS 扩展）**

| PID | 作用 |
| :--- | :--- |
| **SPLIT** | Host→HS Hub 的**分事务令牌**：代 TT 为下游 FS/LS 设备安排 start/complete split（配合 `SS/CS` 字段） |
| **PING** | HS 批量出方向的**空间预约**令牌，设备回 `ACK`（有空间）/`NAK`/`STALL`，避免盲发大包 |
| **ERR** | 分事务中 Hub TT 报告「无法安排 split」，让主机重排期 |
| **PRE** | HS Hub 向下游 LS 设备发**前导（preamble）**，让 LS 设备以 1.5 Mbps 采样后续慢速载荷 |

### 3.3 CRC 校验覆盖范围

| CRC | 多项式 | 初值 | **覆盖字段** | 用在 | 检错能力 |
| :---: | :--- | :---: | :--- | :--- | :--- |
| **CRC5** | $G(x)=x^5+x^2+1$ | 全 1，结果取反 | **PID(4) + ADDR(7) + ENDP(4) = 15 bit**（不含 SYNC、不含 PID 反码） | token / SOF 包 | 全部 ≤2 bit 错、奇数位错、长度 <32 bit 的突发错 |
| **CRC16** | $G(x)=x^{16}+x^{15}+x^2+1$ | 全 1，结果取反 | **DATA 字段全部字节**（不含 SYNC、不含 PID） | data 包 | 同上，突发错长度 <16 bit |

> [!TIP] PID 双保险
> 同一 4 bit PID 被编码两遍（正码 + 反码）。任何一位出错都会破坏互补关系 → 接收方立即判废，且**不产生任何 handshake**，主机因此必然超时重试，这正是"包丢但数据不会串"的原因。

---

## 🔁 4. 事务与四种传输类型

### 4.1 事务（Transaction）= token + data + handshake

```text
OUT 事务（主机→设备）：   [Token: OUT] ──► [Data: DATA0/DATA1] ──► [Handshake: ACK|NAK|STALL]
IN  事务（设备→主机）：   [Token: IN ] ◄── [Data: DATA0/DATA1] ◄── [Handshake: ACK(由主机发)]
SETUP事务（控制建立）：   [Token: SETUP] ─► [Data: 8 B, 固定 DATA0] ─► [Handshake: ACK]
```

| 阶段 | 谁发起 | 内容 | 出错处理 |
| :--- | :---: | :--- | :--- |
| **Token** | Host | 声明「对哪个设备/端点、读还是写、什么类型」 | CRC5 错 → 设备静默丢弃，主机超时 |
| **Data** | Host(OUT/SETUP) 或 Device(IN) | 载荷 + CRC16；**长度决定事务何时结束** | CRC16 错 → 接收方不回 ACK，或回 NAK/STALL |
| **Handshake** | 接收方(OUT) / 发送方确认(IN) | ACK/NAK/STALL/NYET | NAK → 重试；STALL → 需主机干预清除 |

> **一个"传输（Transfer）"可包含多个事务**：例如 1 KB 批量 IN 传输 = 2 个 512 B 事务；HS 批量一包 512 B，**短包（$<$ wMaxPacketSize）或零长包表示传输结束**。

### 4.2 端点、管道与 EP0

| 概念 | 说明 |
| :--- | :--- |
| **Endpoint（端点）** | 设备侧的 FIFO 挂点，方向固定（EP0 例外）；每设备最多 **16 IN + 16 OUT**，编号 4 bit（0~15） |
| **Pipe（管道）** | 主机端点缓冲区 ↔ 设备端点之间的逻辑单工通道，分**流管道（stream，无包边界、无重试：等时）**与**消息管道（message，有包边界、可重试：控制/批量/中断）** |
| **EP0** | **双向**默认控制端点，唯一在枚举未开始就可用的端点；承载 `SETUP/DATA/STATUS`；`bMaxPacketSize0` 决定单事务数据大小 |
| **地址 + 端点** | 包里的 `ADDR(7)+ENDP(4)` 唯一定位一个端点；未配置的设备只在地址 0 响应 |
| **Data Toggle** | 端点维护 DATA0/DATA1 交替状态，保证「同一 data 包不被重复处理」；**NAK 重传时不翻转，ACK 后翻转**；SETUP 强制 DATA0 |

### 4.3 四种传输类型对比（必考）

| 维度 | **控制 Control** | **批量 Bulk** | **中断 Interrupt** | **等时 Isochronous** |
| :--- | :--- | :--- | :--- | :--- |
| 别名 | BTP 消息管道 | 请求/响应管道 | 轮询式小消息 | 流管道 |
| 方向 | 双向（EP0） | 单向（每方向一端点） | 单向 | 单向 |
| 事务组成 | **SETUP + DATA* + STATUS** | IN/OUT 多事务 | IN（或 OUT）单事务 | IN/OUT，**无 handshake** |
| 错误重试 | 有（硬件自动） | 有 | 有 | **无**（丢了就丢了，保时序） |
| 单包最大 | FS 8/16/32/64；HS 64 B | FS 8/16/32/64；HS **512 B** | FS 1~1023；HS 1~1024 B | FS 1023；HS 1024 B（高带宽 3×/µf） |
| 轮询周期 | 立即（`bInterval=0/1`） | 有带宽就用 | `bInterval` 指定（FS 帧、HS 微帧） | `bInterval=1`，**每帧/微帧固定占位** |
| 带宽保证 | 保留 | **尽力而为，最后分配** | 保留（预留） | **优先保留**（硬实时） |
| 典型设备/用途 | 枚举、`GET_DESCRIPTOR`、类命令 | U 盘、打印机、CDC 收发数据 | 键鼠、Hub 状态变化、HID 报告 | 音频/视频、USB 电话 |
| 是否可 Stall | 可 | 可 | 可 | 语义上不用 NAK（用 `DATA2/MDATA` 表错） |

> [!IMPORTANT] 选型口诀
> **枚举只用控制传输；大数据只用批量；小而有界延迟用中断；实时流用等时。**
> 常见误用：把键鼠做成批量 → 延迟不保证；把音频做成中断 → 偶发 NAK 造成爆音（等时允许丢帧但不能丢时序）。

### 4.4 控制传输的三段式（含状态阶段）

```text
① SETUP   Host→EP0: 8 B 标准/类/厂商请求（DATA0），Device 回 ACK
② DATA*   IN : Device→Host  描述符/数据，主机 ACK（DATA0/DATA1 交替）
            OUT: Host→Device 数据，设备回 ACK，末包用零长包收尾
③ STATUS  IN : Host 发 IN，Device 回 0 长度 DATA1，主机 ACK；或反向 OUT 零包
            （方向与 ② 相反，0 长度、无 payload，标志该请求彻底完成）
```

---

## 🚀 5. 枚举流程（Enumeration，逐步）

### 5.1 时序步骤

| # | 动作 | 发起方 | 关键细节 | 设备状态变化 |
| :--: | :--- | :---: | :--- | :--- |
| 1 | **连接检测** | Device | 设备上拉 $1.5\text{ k}\Omega$（D+ = FS/HS、D− = LS）；主机轮询 `port status change (CC)` | Attached → Powered（**电流限 $100\text{ mA}$ 以内**） |
| 2 | **去抖** | Host | 主机等待 $\approx 100\text{ ms}$ 才确认（过滤插拔抖动），Hub 上报 `C_CONNECT` | — |
| 3 | **复位** | Host | Hub 端口置 SE0 **$\ge 10\text{ ms}$**（HS 另有 Chirp K/J 握手升速） | → **Default**（地址 = 0，`bMaxPacketSize0` 生效） |
| 4 | **GET_DESCRIPTOR(device, 8 B)** | Host | 在**地址 0** 上取头 8 字节，只为拿到 `bMaxPacketSize0` | Default |
| 5 | **SET_ADDRESS(n)** | Host | SETUP 数据阶段后由 STATUS 阶段生效；地址 $n \in 1..127$ | → **Address** |
| 6 | **GET_DESCRIPTOR(device 18 B / config / 其他速度)** | Host | 取完整设备描述符 → 取配置描述符头 9 B 得 `wTotalLength` → 再取完整配置块（含接口/端点/类描述符） | Address |
| 7 | **GET_DESCRIPTOR(STRING)** | Host | 先取 `LANGID`（索引 0），再取厂商/产品/序列号；**UTF-16LE** | Address |
| 8 | **SET_CONFIGURATION(c)** | Host | 选择 `bConfigurationValue`；设备据此分配带宽、使能端点，类驱动开始 probe | → **Configured** |
| 9 | **正常数据传输** | Host | 按 `bInterval` 轮询 IN 端点、调度 OUT；HID 报告 / MSD SCSI / CDC 数据开始流动 | Configured |

设备状态机：**Attached → Powered → Default → Address → Configured**，旁挂 **Suspended**（3 ms 无总线活动自动进入，见 §7.4）。

### 5.2 标准设备请求表（`bRequest`）

| 值 | 请求 | 方向 | 作用与典型参数 |
| :--: | :--- | :---: | :--- |
| 0x00 | **GET_STATUS** | D→H | 设备/接口/端点状态；设备返回 bit0 自供电、bit1 远程唤醒；端点返回 bit0 Halt |
| 0x01 | **CLEAR_FEATURE** | H→D | 清除特性：`0`=ENDPOINT_HALT（**恢复被 STALL 的端点**）、`1`=DEVICE_REMOTE_WAKEUP、`2`=TEST_MODE |
| 0x03 | **SET_FEATURE** | H→D | 置特性（同上枚举），或 `TEST_MODE` 进合规测试 |
| 0x05 | **SET_ADDRESS** | H→D | `wValue` = 新地址；数据/状态阶段为 0 长 |
| 0x06 | **GET_DESCRIPTOR** | D→H | `wValue = (类型<<8) \| 索引`；类型 1 设备/2 配置/3 字符串/4 接口/5 端点/6 限定符/7 其他速度 |
| 0x07 | **SET_DESCRIPTOR** | H→D | 极少使用，多数设备直接 STALL |
| 0x08 | **GET_CONFIGURATION** | D→H | 读当前 `bConfigurationValue`（0 = 未配置） |
| 0x09 | **SET_CONFIGURATION** | H→D | `wValue` = 配置值，`0` = 回到未配置 |
| 0x0A | **GET_INTERFACE** | D→H | 读某接口的 `bAlternateSetting` |
| 0x0B | **SET_INTERFACE** | H→D | 切换备用设置（如音频采样率、CDC 数据率模式） |
| 0x0C | **SYNCH_FRAME** | D→H | 等时端点取同步帧号（等时反馈） |

### 5.3 SETUP 包的 8 字节格式（EP0 通用头）

```text
偏移  字段             位含义
 0    bmRequestType    bit7 方向(0=H→D, 1=D→H) | bit6..5 接收者(0 设备/1 接口/2 端点/3 其他) | bit4..0 类型(0 标准/1 类/2 厂商)
 1    bRequest         请求码（见 §5.2，类定义各自扩展）
 2..3 wValue           参数 1（小端）      4..5 wIndex  参数 2（接口号/端点号/语言 ID）
 6..7 wLength          随后 DATA 阶段期望的字节数（小端）
```

> [!WARNING] 描述符必须 packed + 小端
> 多字节字段一律 **little-endian**；C 结构体必须加 `__attribute__((packed))`/`#pragma pack(1)`。编译器插入的填充字节会直接破坏偏移，表现为「主机读到奇怪的设备类」「枚举卡在 GET_DESCRIPTOR」。

---

## 📄 6. 描述符体系与设备类

### 6.1 设备描述符（`bDescriptorType = 0x01`，18 B）

| 偏移 | 字段 | 长度 | 含义 / 取值建议 |
| :--: | :--- | :--: | :--- |
| 0 | `bLength` | 1 | 固定 `0x12`（18） |
| 1 | `bDescriptorType` | 1 | `0x01` |
| 2 | `bcdUSB` | 2 | `0x0110` FS/LS、`0x0200` HS；`0x0201` 支持 BM（远程唤醒挂起） |
| 4 | `bDeviceClass` | 1 | **`0x00` = 类信息在接口层**（复合设备必须填 0 + 用 IAD）；`0xEF/0x02/0x01` = Other/Composite |
| 5 | `bDeviceSubClass` | 1 | 同上，`0x00` |
| 6 | `bDeviceProtocol` | 1 | 同上 |
| 7 | `bMaxPacketSize0` | 1 | **只能 8/16/32/64**；HS 必须 64；FS 建议 64（枚举包数直接受影响） |
| 8 | `idVendor` | 2 | **必须向 USB-IF 申请**；量产禁自用未授权 VID |
| 10 | `idProduct` | 2 | 厂商自定义 |
| 12 | `bcdDevice` | 2 | 固件版本（BCD） |
| 14/15/16 | `iManufacturer` / `iProduct` / `iSerialNumber` | 1 | **字符串描述符索引**，`0` = 无该字符串 |
| 17 | `bNumConfigurations` | 1 | 配置数（通常 1） |

### 6.2 配置描述符（`0x02`，9 B 头 + 后续全部描述符）

| 字段 | 说明 |
| :--- | :--- |
| `bLength` / `bDescriptorType` | `0x09` / `0x02` |
| `wTotalLength` | **配置块总长**（配置 + 所有接口 + 所有端点 + 类专属描述符）。主机先读 9 B 再按此值重读，**算错必然枚举失败** |
| `bNumInterfaces` | 接口数（= 描述符中"第一组"接口数，不含 alternate 重复计数） |
| `bConfigurationValue` | `SET_CONFIGURATION` 用的值 |
| `iConfiguration` | 字符串索引 |
| `bmAttributes` | bit7 恒 1（保留）· **bit6 = 总线供电（1）/自供电（0）** · bit5 = 远程唤醒 · bit4..0 = 0 |
| `bMaxPower` | **单位 2 mA**：`0x32` = 100 mA，`0xFA` = 500 mA（HS 上限） |

$$
I_{\max} = bMaxPower \times 2\text{ mA} \qquad\text{（自供电设备仍要如实上报总线取电）}
$$

### 6.3 接口描述符（`0x04`，9 B）与端点描述符（`0x05`，7 B）

| 接口字段 | 含义 |
| :--- | :--- |
| `bInterfaceNumber` / `bAlternateSetting` | 接口号 / 备用设置号（同一接口多套端点/参数时切换） |
| `bNumEndpoints` | 该接口下端点数（**不含 EP0**） |
| `bInterfaceClass/SubClass/Protocol` | 类三元组，决定主机加载哪个驱动（见 §6.4） |
| `iInterface` | 字符串索引 |

| 端点字段 | 含义 |
| :--- | :--- |
| `bEndpointAddress` | bit7 = 方向（0 = OUT，1 = IN），bit3..0 = 端点号 |
| `bmAttributes` | bit1..0 传输类型（**00 控制 / 01 等时 / 10 批量 / 11 中断**）；等时再占 bit4..2 同步类型、bit6..5 使用类型 |
| `wMaxPacketSize` | bit10..0 包大小；bit12..11（HS 等时）每微帧**额外事务数**（0 单次 / 1 双次 / 2 三次） |
| `bInterval` | FS：帧数 $1\sim 255$；HS 中断/等时：$2^{(n-1)} \times 125\ \mu s$；HS 批量/控制：忽略 |

### 6.4 常用类（Class Code）速览

| 类 | `bInterfaceClass` | 子类/协议要点 | 端点形态 | 说明（英文） |
| :--- | :--: | :--- | :--- | :--- |
| **HID** | `0x03` | 子类 `0x01` Boot（`0` 键盘 / `1` 鼠标），协议 `0` 无 / `1` 键盘 / `2` 鼠标 | 1 个 **IN 中断** 端点（+ 可选 OUT） | Human Interface Device；需额外 `HID` 描述符 + **Report Descriptor**，主机靠 `GET_REPORT`/中断 IN 收报告 |
| **CDC** | `0x02` | `0x02` ACM（虚拟串口）· `0x06` ECM · `0x0D` NCM | ACM = 控制接口（1 个 IN 中断通知端点） + 数据接口（1 IN + 1 OUT **批量**） | Communications Device Class；**最常用「USB 转串口」免驱方案**，与 [[UART]] 应用层等价，但要两个接口 + IAD/类请求（`SET_LINE_CODING` 等） |
| **MSD / MSC** | `0x08` | 子类 `0x06`（SCSI 透明），协议 `0x50` **Bulk-Only Transport (BOT)** | 1 IN + 1 OUT **批量** | Mass Storage Class；命令为 31 B **CBW** → 数据 → 13 B **CSW**，主机 SCSI 栈重发能力要求设备严格处理阶段错序 |
| **Audio / Printer / Hub** | `0x01 / 0x07 / 0x09` | 等时流 / 双向批量 / 单 IN 中断 | 视类而定 | Hub 必须只有一个中断 IN 端点上报端口变化 |
| **Vendor Specific** | `0xFF` | 自由定义 | 通常批量 | 配 WinUSB/libusb 自定义上位机，协议自己做（可借鉴 [[Modbus]] 式帧 + CRC） |

---

## ⏱️ 7. 帧/微帧、SOF、带宽调度与版本差异

### 7.1 帧与 SOF

| 项目 | FS / LS | HS (USB 2.0) | SS (USB 3.x) |
| :--- | :--- | :--- | :--- |
| 帧（Frame） | $1\text{ ms} \pm 0.5\%$，12000 bit | $1\text{ ms}$（60000 bit） | 无 SOF 帧，改为**链路层 + 包**（ITP 保序） |
| 微帧（Microframe） | 无 | **$125\ \mu s$**，每帧 8 个 | — |
| SOF 内容 | PID + 11 bit 帧号 + CRC5，**广播、无 ACK** | 同（每微帧一次） | 由 Link/Protocol 层管理 |
| 每帧/微帧最多事务 | 1 次/端点/帧 | HS 批量可多事务（MTT）+ split 事务 | 全双工，双通道 |
| LS/FS 挂在 HS Hub 下 | — | 经 TT 做 **start/complete split**（`SPLIT + SS/CS`），HS 侧只占用小段带宽 | 不需要 TT |

### 7.2 带宽估算公式

$$
\text{端点带宽} = \frac{\text{wMaxPacketSize} \times \text{每(微)帧事务次数}}{\text{bInterval 对应时间}}
$$

| 场景 | 计算 | 结果 |
| :--- | :--- | :---: |
| FS 中断 IN，`bInterval=1`，64 B | $64\text{ B} \times 1000/\text{s}$ | $62.5\text{ KB/s}$ |
| HS 中断 IN，`bInterval=4`（$2^3 \times 125\ \mu s = 1\text{ ms}$），64 B | $64 \times 1000$ | $62.5\text{ KB/s}$ |
| HS 批量，512 B、每 µf 1 事务、连续占 8 µf | $512 \times 8 \times 8000$ | $\approx 32.8\text{ MB/s}$ |
| HS 高带宽等时，$3 \times 1024$ B / µf | $3072 \times 8000$ | $\approx 24\text{ MB/s}$ |

- 主机为控制/保留类预留带宽，规范规定**每帧 HS 最多约 90% 用于周期性传输**，批量（非周期）吃剩余；因此"HS 480 Mbps"实测批量吞吐仅约 $35\sim 42\text{ MB/s}$（协议开销 + NAK 重试 + 软件栈）。
- 帧开销示例：SOF 30 bit / FS 帧 12000 bit → $0.25\%$；但 token+handshake 的固定开销使控制传输 8 B 载荷的实测效率常低于 $30\%$。

### 7.3 USB 2.0 与 3.0 差异小表

| 维度 | **USB 2.0 (HS)** | **USB 3.x (SuperSpeed)** |
| :--- | :--- | :--- |
| 速率 | 480 Mbps（半双工，一线共享） | 5 / 10 Gbps per lane，**双通道全双工** |
| 差分线对 | 1 对（D+/D−） | 另加 **SSTX± + SSRX±** 两对，D+/D− 仍用于 2.0 兼容枚举 |
| 编码 | NRZI + 位填充（自时钟） | **8b/10b**（Gen1）/ **128b/132b**（Gen2）+ SKP 有序集补偿时钟差 |
| 传输模型 | Host 轮询 token，事务三段式 | **链路层（TLP/Packet header）+ 流（stream）+ No-Op/ITP**；设备可主动通知（U1P/LGO） |
| 端点定义 | `wMaxPacketSize` + `bInterval` | 新增 **SS Endpoint Companion（类型 `0x30`）**： bursts + 每 burst 包数 |
| 电源状态 | Attached / Default / Address / Configured + Suspended | 链路状态 **U0 / U1 / U2 / U3**（U3 = 挂起，用 **LFPS** 握手唤醒） |
| 供电上限 | 500 mA / 口（HS 单位负载） | **900 mA**；BC1.2 可 1.5 A（D+/D− 短接或分压识别） |
| Hub | TT 做 LS/FS 分事务 | 根/Hub 需同时管理 2.0 与 SS 两棵拓扑（同一设备两条链路） |

### 7.4 常见坑与排查（现场向）

> [!CAUTION] 挂起（Suspend）
> **总线连续 3 ms 无 SOF/活动** → 设备硬件自动进 Suspended，并把取电降到 FS $500\ \mu A$ / HS $2.5\text{ mA}$。此时若固件不处理 `RESET/SUSPEND` 事件：① 唤醒后状态机不一致、② 主机看到「设备消失后重现」反复枚举。远程唤醒需 `bmAttributes` bit5 + `SET_FEATURE(DEVICE_REMOTE_WAKEUP)`，恢复信令为 **resume signaling（J 态持续 $\ge 20\text{ ms}$）**。

> [!FAILURE] 枚举失败排查顺序（由物理到逻辑）
> 1. **无「叮」声/无端口事件** → 上拉错（D+/D− 接反、$1.5\text{ k}\Omega$ 漏焊）、VBUS 未到位、GND 不共地、Type-C 无 CC 下拉。
> 2. **`device descriptor read/64, error -71 (EPROTO)`** → 信号质量：走线阻抗非 $90\ \Omega$ 差分、ESD 电容过大、HS 未做阻抗控制/过长；换短线/降 HS 验证。
> 3. **`unable to enumerate USB device` / `-110 (ETIMEDOUT)`** → 设备未响应 EP0：EP0 中断没开、`bMaxPacketSize0` 与实现不符、复位后未回 Default、SET_ADDRESS 前用了错误地址。
> 4. **反复 `-32 (EPIPE)`/端点 STALL** → 请求了未实现的功能或数据翻转（toggle）不同步；**`CLEAR_FEATURE(ENDPOINT_HALT)` 无效即为主因**：检查 toggle 只在 ACK 后翻转。
> 5. **描述符读回长度对不上** → `wTotalLength` 手填错误、结构体被对齐填充、字符串忘加 `0x0409` LANGID 或不是 UTF-16LE。
> 6. **枚举 OK 但无驱动/无数据** → `bInterfaceClass` 三元组不匹配类驱动、缺 IAD（复合设备）、`SET_CONFIGURATION` 后端点未使能、`bInterval` 填 0 导致主机不轮询。
> 7. **HS 设备被判成 FS** → Chirp 握手失败（HS PHY 未供电/未校准）、Host 端 TT 问题；`lsusb -t` 会显示 `12M/480M`。
> 抓包定位：Linux `usbmon` + Wireshark（`usb.addr == 3.0`、过滤 `usb_packet`）、Windows USBPcap/BusHound；内核侧 `dmesg`、`lsusb -v -d VID:PID`、`lsusb -t`。

---

## 🖼️ USB 包结构与枚举时序图

![[USB.excalidraw]]

> [!TIP] 图看点
> - **左画布**：SYNC→PID→ADDR/ENDP→DATA→CRC→EOP 的字段相邻关系与位宽；下方展开 **OUT 与 IN 两种 token** 的 8 bit PID（4 位类型 + 4 位反码）与 CRC5 覆盖范围。
> - **右画布**：Host/Device 两条生命线上的枚举消息序列 —— 插拔检测 → 复位 → `GET_DESCRIPTOR(8B)` → `SET_ADDRESS` → 取完整描述符 → `SET_CONFIGURATION` → 正常数据通信，并标注每步对应的设备状态。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] USB 设备固件/硬件上电前必过
> - [ ] **上拉与速率**：FS/HS 设备 D+ 挂 $1.5\text{ k}\Omega$，LS 设备挂 D−；未连设备两线均被主机 $15\text{ k}\Omega$ 下拉
> - [ ] **供电与时序**：VBUS $5\text{ V} \pm 5\%$、双端共地；枚举完成前取电 $\le 100\text{ mA}$，`bMaxPower` 如实上报（单位 2 mA）
> - [ ] **信号完整性**：D+/D− 差分 $90\ \Omega$ 阻抗、等长成对、换层少；HS 才需严格控制，ESD/共模器件结容量不能超标
> - [ ] **EP0 正确**：`bMaxPacketSize0` ∈ {8,16,32,64} 且与实际 FIFO 一致；SETUP **无条件接受**且强制 DATA0
> - [ ] **描述符自检**：各 `bLength` 正确、`wTotalLength` = 实际字节数、多字节小端、结构体 `packed`、字符串 UTF-16LE + LANGID
> - [ ] **状态机**：复位 → Default（地址 0）→ `SET_ADDRESS` → Addressed → `SET_CONFIGURATION` → Configured；未配置前不响应非 EP0 端点
> - [ ] **数据翻转**：每个非 SETUP 端点维护 DATA0/DATA1，**仅 ACK 后翻转**；重发不重复翻转
> - [ ] **流控语义**：暂时没数据/没空间回 **NAK**，永久错误回 **STALL**（并支持 `CLEAR_FEATURE` 解除），别用 NAK 掩盖故障
> - [ ] **SOF/挂起**：处理 SOF、Suspend（3 ms 无活动）与 Resume；需要时声明并实现远程唤醒
> - [ ] **类匹配**：`bInterfaceClass/SubClass/Protocol` 三元组与目标驱动一致；复合设备补 IAD（`0x0B`）
> - [ ] **抓包验证**：`dmesg` + `lsusb -t/-v` 或 Wireshark+usbmon 完整看到枚举序列，四种传输类型分别做压力/短包/边界用例
> - [ ] **合规**：VID 授权、（量产）USB-IF 电气/协议兼容性测试（眼图、`bMaxPower`、suspend/resume 用例）

---

## 🔗 思考与延伸索引

- [[UART]] — 同为"无时钟线"串行总线：UART 靠约定波特率 + 起始/停止位成帧，USB 靠 NRZI + 位填充 + SYNC/EOP 成帧
- [[SPI]] — 有 SCK 的同步主从总线对照：片选即寻址、无协议层检错；USB 用 ADDR+ENDP 与 CRC 换取共享长线缆能力
- [[PCLe]] — 同样是"差分对 + 编码 + 链路层 + 枚举（BIOS 配置空间）"的高速串行互联，可与 USB 的枚举/描述符思想对比
- [[Modbus]] — 应用层协议视角：Modbus 用 CRC-16 与地址字段自证，USB 把检错（PID 反码 / CRC5 / CRC16）做进每一类包
- [[CAN]] — 总线仲裁/多主 vs USB 单主轮询：CAN 靠 ID 优先级竞争，USB 完全由 Host 排期，无冲突概念

---

## 📚 参考资料

1. **USB-IF** — *Universal Serial Bus Specification, Revision 2.0*（2000；§6 描述符、§7 事务与包、§8 电气与信号编码、Table 8-6 PID、Table 9-4 标准请求）.
2. **USB-IF** — *Universal Serial Bus 3.2 Specification, Release 2*（2017；SuperSpeed  PHY/链路层、8b/10b 与 128b/132b、U0~U3 电源状态）.
3. **USB-IF** — *Universal Serial Bus Class Definitions for Communication Devices, Rev 1.2*（CDC：ACM/ECM/NCM 接口组合与类请求）.
4. **USB-IF** — *Device Class Definition for Human Interface Devices (HID), Rev 1.11* + *Device Class Definition for Physical (PVT) Devices / HID 1.1*（Report Descriptor 与描述符类型 `0x21/0x22`）.
5. **USB-IF** — *Mass Storage Class Bulk-Only (BBB) Transport Specification, Rev 1.0*（CBW/CSW 与 SCSI 透明命令集）.
6. **Jan Axelson** — *USB Complete: The Developer's Guide*, 5th ed., Microsoft Press, 2015（枚举流程、描述符实战与坑的权威工程参考）.
7. **Maxim/Intel 等** — *USB 硬件设计指南（差分阻抗、上拉检测、ESD 与挂起电流）*；Linux 内核文档 `Documentation/usb/`（`usbmon`、错误码与枚举日志解读）.
