---
title: PCIe 总线协议完整技术手册
tags:
  - Embedded
  - Protocol
  - PCIe
  - LTSSM
  - TLP
  - FlowControl
status: Completed
created: 2026-09-21
---

# 🖥️ PCIe 总线协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：PCI Express（Peripheral Component Interconnect Express）是**点对点、串行、基于分组（packet-based）**的高速互连。逻辑上只有事务层 / 数据链路层 / 物理层三层；并行 PCI 的"共享总线 + 地址数据复用"被彻底替换为"交换式拓扑 + 打包传输"，可靠性靠 **CRC + 序号 + ACK/NAK 重传 + Credit 流控**在链路本地闭环完成。
> - **内容范围**：分层架构与差分接线 · 各代速率与编码开销 · 带宽公式 · TLP/DLLP/Ordered Set 包格式 · LTSSM 链路训练 · 流控与重传 · 配置空间/BAR/枚举 · 中断 · 电源管理
> - **推荐路径**：§1 分层与接线 → §2 速率与编码 → §3 包格式（核心）→ §4 LTSSM → §5 可靠性机制 → §6 配置与枚举 → §7/§8 中断与功耗
> - **规范口径**：本文以 PCIe Base Specification 4.x/5.x 为描述基准；个别字段位号在不同修订版有细微调整，落地时以所用版本原图为准（Gen6 的 PAM-4/FLIT 只在 §2 做对照说明）。

---

## 🏗️ 1. 分层架构与链路组成

### 1.1 三层模型（Transaction / Data Link / Physical）

| 层 | 英文 | 核心职责 | 对应包 / 传输单元 | 关键机制 |
| :--- | :--- | :--- | :--- | :--- |
| **事务层** | Transaction Layer (TL) | 把读写请求/完成/消息**组装成 TLP**、把收到的 TLP 拆回内存/IO/配置读写语义；地址空间与排序 | **TLP**（3 或 4 DW 头 + 0~1024 DW 数据） | 4 种地址空间（Memory / IO / Configuration / Message）、排序规则、**Credit 申请与检查**、TC/Attr、ECRC 插入位（TD） |
| **数据链路层** | Data Link Layer (DL) | **链路本地可靠传输**：给 TLP 加序号与 LCRC、组织 ACK/NAK、重传、维护流控信用计数 | **DLLP**（Ack/Nak、FC Update、PM、Link/Vendor）+ 被封装的 TLP | SeqNum[11:0]、**LCRC(32b)**、Retry Buffer、InitFC/UpdateFC、电源状态握手 |
| **物理层** | Physical Layer (PL) | 逻辑子层（加扰、编码、对齐、SKP 插入、Ordered Set、**LTSSM**）+ 电气子层（SerDes、PLL/CDR、驱动/接收、均衡） | **Symbol / Ordered Set**（TS1、TS2、SKP、SDS、EIEOS…） | 8b/10b → 128b/130b → 1b/1b(PAM-4)、lane 倒换与极性反转、CDR 锁定、预设/均衡 |

> [!NOTE] 分层归属易错点
> - **LTSSM 属于物理层**，不属于链路层；但"链路训练成功"这一结果被上层（DL 的 Credit 初始化）用作发送前提。
> - **流控是端到端（一条 Link 的两端）而非全网端到端**：Credit、LCRC、重传都只在相邻两个 Port 之间有效，跨 Switch 会在每个桥端口重新打包，因此"可靠"是按 hop 保证的。
> - 事务层没有介质访问控制/仲裁——点对点链路上发送权天然唯一，这是 PCIe 敢于去掉 PCI 仲裁器的根本原因。

### 1.2 Link、Lane 与差分对

| 概念 | 英文 | 说明 |
| :--- | :--- | :--- |
| **差分对** | Pair | 1 对（P/N 两根线）承载一个方向的串行比特流 |
| **通道** | Lane | **1 发 + 1 收 = 2 对 = 4 根线**，全双工 |
| **链路** | Link | 若干 Lane 聚合：x1 / x2 / x4 / x8 / x16 / x32，各 Lane 独立训练后绑定 |
| **端口** | Port | 链路的端点：Root Port、Switch Up/Downstream Port、Endpoint |
| **端口扩展** | Lane 编号 / 反向 | Lane 编号 0…N-1；支持 **Lane Reversal**（物理插头反插）与 **P/N Polarity Inversion**（差分极性反接），接收端自动纠正 |

**交叉连接关系（与 UART 同为"发对收"，但 PCIe 是差分且各 Lane 独立）：**

```text
        Root Complex / Host Port                         Device / Endpoint Port
   ┌───────────────────────────┐                     ┌───────────────────────────┐
   │                    TX Lane0│══ P/N 差分对 ══════►│Rx Lane0                    │  Host → Dev
   │                    TX Lane1│══ P/N 差分对 ══════►│Rx Lane1                    │  (下行方向)
   │                            │                     │                            │
   │                    RX Lane0│◄══ P/N 差分对 ══════│Tx Lane0                    │  Dev → Host
   │                    RX Lane1│◄══ P/N 差分对 ══════│Tx Lane1                    │  (上行方向)
   └───────────┬───────────────┘                     └───────────┬───────────────┘
               │  REFCLK 100 MHz ±300 ppm（共时钟/独立时钟）      │
               └──────────────────── GND（差分参考地）───────────┘
   另：PERST#（复位）、CLKREQ#（参考时钟电源管理）、WAKE#（可选，开漏）
```

- **速率与线对数**：x4 链路 = 8 个差分对 = 16 根信号线，另加参考时钟与边带信号；对比 PCI 32 bit @33 MHz 需 32 条地址/数据 + 十余条控制线，PCIe 用极少的线换来数量级带宽。
- **AC 耦合**：每对信号线串联 **176~265 nF** 耦合电容（Gen3+ 常用 220 nF），放在**发送端一侧**（规范约束：靠近 TX 引脚），用于隔离两端共模电平。
- **阻抗**：差分 **85 Ω（移动/CEM 常用 100 Ω 名义值）**，单端约 40~50 Ω；走线必须等长匹配并遵守对内 skew 上限（Gen5 对内 skew 通常要求 < 几 ps 量级，见规范电气表）。

> [!WARNING] PCIe 不是总线，不能"并挂"
> 一条 Link 只允许**两个 Port**。要接多个设备必须用 **Switch（虚拟 PCI-PCI 桥级联）** 或在 BIOS/UEFI 里做多 Root Port 分叉；这是与 [[I2C]]、[[Modbus]] 之类多主/多从总线的根本差异。

### 1.3 与并行 PCI 的概念映射

| PCI / PCI-X（并行） | PCIe（串行分组） |
| :--- | :--- |
| 地址 + 数据复用同一组线 | 地址与数据都装进 **TLP**，在不同时刻串行传输 |
| 共享总线 + 仲裁器 + 帧（FRAME/IRDY） | 点对点 + **交换机路由**（按地址/Bus 号） |
| 总线频率统一（33/66/133 MHz） | 每代 **GT/s**（比特率），带宽随 Lane 数线性扩展 |
| 无链路层校验（依赖奇偶/总线时序） | **LCRC + 序号 + ACK/NAK 重传 + Credit**，信号差也不会静默出错 |
| `AD[31:0]` 上直接采样驱动 | 边带信号只剩 PERST#/CLKREQ#/WAKE#，中断/握手全部走消息包（INTx→Msg、MSI→MWr） |

---

## ⚡ 2. 电气、编码与带宽

### 2.1 各代速率与信号方式

| 代次 | 年份 | 每 Lane 速率 | 信号 | 线路编码 | UI（比特/符号周期） | 每 Lane 有效带宽（单向） |
| :--: | :--: | :--: | :--: | :--- | :--: | :--: |
| **Gen1** | 2003 | **2.5 GT/s** | NRZ | 8b/10b | 400 ps | 250 MB/s |
| **Gen2** | 2007 | **5.0 GT/s** | NRZ | 8b/10b | 200 ps | 500 MB/s |
| **Gen3** | 2010 | **8.0 GT/s** | NRZ | 128b/130b | 125 ps | ≈ 984.6 MB/s |
| **Gen4** | 2017 | **16.0 GT/s** | NRZ | 128b/130b | 62.5 ps | ≈ 1 969 MB/s |
| **Gen5** | 2019 | **32.0 GT/s** | NRZ | 128b/130b | 31.25 ps | ≈ 3 938 MB/s |
| **Gen6** | 2022 | **64.0 GT/s** | **PAM-4**（32 GBaud） | 1b/1b + **FLIT + FEC** | 31.25 ps/符号（15.6 ps/比特） | ≈ 7 375 MB/s |

> [!WARNING] 代次与速率的对应最容易记错
> `2.5 / 8 / 16 / 32 / 64 GT/s` 分别是 **Gen1 / Gen3 / Gen4 / Gen5 / Gen6**——中间还有一个常被漏掉的 **Gen2 = 5.0 GT/s**，而 **64 GT/s 属于 Gen6（PAM-4）**，不是 Gen5。做驱动/PHY 配置时以 `PCFG`/速率协商结果为准，不要靠"第几代"的口语推断。

**其他共性电气参数（规范约束，工程取值参考）**

| 项目 | 典型值 / 说明 |
| :--- | :--- |
| 参考时钟 | **100 MHz**；共时钟（SRNS）两端各 ±300 ppm → 合计容差 600 ppm；独立扩频时钟 SRIS 各 ±300 ppm |
| AC 耦合电容 | 176 ~ 265 nF（Gen3+ 典型 220 nF），置于发送侧 |
| 差分阻抗 | 85 Ω / 100 Ω 名义值，对内 skew 严格受限 |
| 信道损耗预算 | Gen3 ≈ 8~9 dB @ 4 GHz；Gen4 ≈ 16 dB @ 8 GHz；Gen5 ≈ 28~36 dB @ 16 GHz（含封装/过孔，**以 Base Spec 与 CEM 表为准**） |
| 发送均衡 | Gen1/2 仅去加重（−6/−3.5 dB）；**Gen3+ 为 3 抽头 FIR（pre/main/post）**，预设 P0~P10 + 接收端系数请求（EQ Phase 0~3） |
| 接收均衡 | CTLE → DFE（Gen4/5 普遍多抽头 DFE），CDR 恢复比特率 |
| 发送端检测 | Receiver Detect：通过阻抗/充放电判断对端是否存在（Detect 状态的基础） |

### 2.2 编码开销对比：8b/10b vs 128b/130b vs FLIT

| 对比项 | **8b/10b**（Gen1/Gen2） | **128b/130b**（Gen3/4/5） | **1b/1b + FLIT/FEC**（Gen6） |
| :--- | :--- | :--- | :--- |
| 每传输单元 | 10 bit 携带 8 bit 有效 | 130 bit 携带 128 bit 有效（+2 bit block type） | 固定 **256 B FLIT** |
| 编码效率 | **80%** | **≈ 98.46%** | 236/256 ≈ **92.2%**（TLP 载荷，扣 DLP/CRC/FEC） |
| 额外开销 | 25% | ≈ **1.56%** | FEC+CRC 约 7.8%（另有按代次变化的 DLP 字段） |
| 直流平衡 | 依赖 **Running Disparity**（RD+/RD−） | 由**加扰**保证 | 由加扰保证 |
| 符号同步 | 依赖 **COMMA（K28.5）** 检测做字节对齐 | 无需 COMMA，靠 **block type + bit lock + SDS** 对齐 | 靠 FLIT 边界 + sequence 对齐 |
| 加扰 LFSR | 16 bit 多项式 $x^{16}+x^5+x^4+x^3+1$ | 23 bit LFSR，**每 Lane 不同种子** | 23 bit LFSR 系列（含 Retimer 相关） |
| 错误可检性 | 非法码组（disparity/K 码违规）可直接发现 | 主要靠 LCRC/Framing 检查 | **FEC 前向纠错 + CRC 检错 + 重传** |
| 眼图余量 | UI 大、均衡简单 | UI 缩小 1.6×，需 DFE | PAM-4 三电平 → 幅度余量降至约 1/3，必须 FEC |

> [!ABSTRACT] 为什么 Gen3 一定要换编码
> 8 GT/s 下若仍用 8b/10b，25% 开销等价于要求前端多跑 25% 的比特率（10 GBaud），信道损耗和均衡压力无法承受；改用 128b/130b 把开销压到 1.5%，代价是**失去 COMMA/K 码这种逐符号边界信息**——所以 Gen3+ 必须用 block（含 block type 位）与 **SDS Ordered Set** 完成对齐，并把"是否出现乱序/丢块"的检查责任完全交给 **LCRC + Framing Token**。

### 2.3 带宽公式与各代实速表

**每 Lane、单向有效带宽：**

$$
BW_{\text{lane}} = R_{\text{GT/s}} \times \eta_{\text{编码}} \times \frac{1}{8} \quad \left[\frac{\text{GB}}{\text{s}}\right]
$$

**整链路（N 条 Lane，单向 / 双向）：**

$$
BW_{\text{单向}} = \frac{R \times N \times \eta}{8},\qquad
BW_{\text{双向}} = 2 \times BW_{\text{单向}},\qquad
\eta_{8b/10b}=0.8,\quad \eta_{128b/130b}=\frac{128}{130}\approx 0.9846
$$

以 Gen3（$R=8$，$\eta=0.9846$）x4 为例：

$$
BW_{\text{单向}} = \frac{8 \times 4 \times 0.9846}{8} = 3.94\ \text{GB/s} \quad(\text{双向} \approx 7.88\ \text{GB/s})
$$

**单向有效带宽表（GB/s，已扣除编码开销；双向值 ×2）**

| 代次 / 编码                          |    x1 |    x2 |    x4 |    x8 |   x16 |
| :------------------------------- | ----: | ----: | ----: | ----: | ----: |
| **Gen1** 2.5 GT/s（8b/10b）        |  0.25 |  0.50 |  1.00 |  2.00 |  4.00 |
| **Gen2** 5.0 GT/s（8b/10b）        |  0.50 |  1.00 |  2.00 |  4.00 |  8.00 |
| **Gen3** 8.0 GT/s（128b/130b）     | 0.985 |  1.97 |  3.94 |  7.88 | 15.75 |
| **Gen4** 16.0 GT/s（128b/130b）    |  1.97 |  3.94 |  7.88 | 15.75 |  31.5 |
| **Gen5** 32.0 GT/s（128b/130b）    |  3.94 |  7.88 | 15.75 |  31.5 |  63.0 |
| **Gen6** 64.0 GT/s（PAM-4 + FLIT） |  7.38 | 14.75 |  29.5 |  59.0 | 118.0 |

**再扣 TLP/DLLP 开销后的实测吞吐（估算，用于容量规划）**

| 场景               | 单包开销构成（约）                                             |     载荷效率      |         Gen3 x4 实际吞吐（约）         |
| :--------------- | :---------------------------------------------------- | :-----------: | :-----------------------------: |
| MWr，MPS = 256 B  | 头 12 B + DLL 序号 2 B + LCRC 4 B + STP/END 2~4 B ≈ 20 B | 256/276 ≈ 93% |            3.65 GB/s            |
| MWr，MPS = 128 B  | 同上 ≈ 20 B                                             | 128/148 ≈ 86% |            3.40 GB/s            |
| MRd，MRRS = 512 B | 读请求只有头；完成按 MPS 切包 + 每包头/ECRC                          |   与 MPS 强相关   |          3.3~3.7 GB/s           |
| 小包（寄存器 4 B 读写）   | 头占绝对主导                                                |     < 20%     | < 0.8 GB/s，且受 Credit/Latency 限制 |

> [!TIP] 带宽规划三步法
> 1. 先算**编码后物理带宽**（本节表格）；2. 再按 **MPS / MRRS** 算 TLP 头与 LCRC 摊薄比例；3. 最后检查是否被 **Credit 数量、Outstanding Tag 数、往返延迟**卡住（`实际带宽 ≤ Credit 数据量 / 往返延迟`，见 §5.1 与 [[DMA、Buffer 与流量控制]]）。

---

## 📦 3. 包格式：TLP / DLLP / Ordered Set

### 3.1 TLP 在线路上的完整封装

```text
 ┌─────┬──────────────┬───────────────────────┬──────────────────┬───────┬───────┬─────┐
 │ STP │ DLL Header   │  TLP Header           │  Data Payload    │ ECRC  │ LCRC  │ END │
 │Token│ 2B：SeqNum[11:0] │ 3 DW（32b 地址）/4 DW（64b 地址） │ 0 ~ 1024 DW（≤ 4 KB） │ 4B(选)│ 4B    │Token│
 └─────┴──────────────┴───────────────────────┴──────────────────┴───────┴───────┴─────┘
   PL    ← 数据链路层添加：只在本段 Link 有效  →        ← 事务层生成 →    ← 端到端 →  ← 链路本地 →  PL
```

- **STP / END**：分界 Token（8b/10b 下为特殊 K 码组合；128b/130b 下由 Framing Token 表达），Gen6 FLIT 模式下由 FLIT 结构替代。
- **LCRC（32 bit CRC）**：覆盖从 STP 之后到 LCRC 之前的全部位，**链路本地**（每个 Switch Port 重新计算），检错后触发 NAK 重传。
- **ECRC（32 bit，可选，由 TD 位标记）**：由事务层产生、**端到端**（源 → 最终目标）校验，Switch 不重算，可发现桥内数据损坏。
- **SeqNum[11:0]**：数据链路层的 12 bit 循环序号，是 Ack/Nak 与重传缓冲的索引（不属于事务层语义）。

### 3.2 TLP 通用头 DW0（所有 TLP 共享）

> 位号以 MSB→LSB 排列给出；不同修订版在 Tag 扩展（8/10 bit）与 TH 引入时对 R/Attr 位有细微挪动，字段**位宽**不变。

| 字段              | 位宽  | 位置         | 说明                                                                               |
| :-------------- | :-: | :--------- | :------------------------------------------------------------------------------- |
| **Fmt[2:0]**    |  3  | DW0[31:29] | 包格式：**是否指定长度 / 3 DW 还是 4 DW（32/64 位地址）/ 是否带数据**；由此区分 MRd32/MRd64/MWr32/MWr64 等   |
| **Type[5:0]**   |  6  | DW0[28:23] | 事务类型：Memory / IO / Cfg / Msg / Cpl（含 AtomicOp 等扩展），详见 §3.4                       |
| **R / T9**      |  1  | DW0[22]    | 10-bit Tag 的最高位；未启用时为保留位                                                         |
| **TC[2:0]**     |  3  | DW0[21:19] | Traffic Class，**默认 0**；非 0 需两端虚拟通道（VC）仲裁支持，Gen6 FLIT 模式不再使用 TC                   |
| **R**           |  1  | DW0[18]    | 保留                                                                               |
| **R / T8**      |  1  | DW0[17]    | 10-bit Tag 的第 8 位；未启用时保留                                                         |
| **TD**          |  1  | DW0[16]    | **TLP Digest**：1 = 尾部带 ECRC                                                      |
| **EP**          |  1  | DW0[15]    | 请求类：**1 = Poisoned TLP（数据被污染）**；完成类：1 = `Status`/Error 字段有效（如 UR、CA）             |
| **Attr[1:0]**   |  2  | DW0[14:13] | Attr[1] = **RO（Relaxed Ordering）**，Attr[0] = **RCB（Read Boundary）**，用于放宽排序/提升读合并 |
| **TH**          |  1  | DW0[12]    | **TPH（TLP Processing Hints）** 存在标志，携带 ST（Steering Tag）提示缓存放置                     |
| **AT[1:0]**     |  2  | DW0[11:10] | Address Type：地址翻译相关（**ATS** 未翻译/已翻译地址、TLP 前缀）                                    |
| **Length[9:0]** | 10  | DW0[9:0]   | 数据长度，单位 **DW（4 B）**；`0` 表示 **1024 DW = 4096 B**；受 MPS/MRRS 约束                    |

### 3.3 地址头与 ID/Tag（DW1~DW3）

**3 DW 头（32 位地址，如 MRd32 / MWr32 / IORd / CfgRd）**

| DW  | 字段                              |  位宽  | 说明                                                        |
| :-: | :------------------------------ | :--: | :-------------------------------------------------------- |
| DW1 | **Address[31:2]**               |  30  | 双字对齐的 32 位地址（低 2 位恒 0；DW 边界对齐由 Length 与 BE 决定）            |
| DW2 | **Requester ID[15:0]**          |  16  | Bus[15:8] : Device[7:3] : Function[2:0]，即 RID             |
| DW2 | **Tag[7:0]**（或 +T9/T8 → 10 bit） | 8/10 | 未完成事务标记，用于把 Cpl 匹配回请求；决定 outstanding 上限（8 bit → 256 个并发读） |
| DW2 | **TH/ST 或 R**                   |  —   | TH=1 时此处放 Steering Tag                                    |
| DW2 | **Last DW BE[3:0]**             |  4   | 末双字字节使能（每 bit 对应 1 字节）                                    |
| DW2 | **1st DW BE[3:0]**              |  4   | 首双字字节使能                                                   |

**4 DW 头（64 位地址，MRd64 / MWr64）**：DW1 = **Address[63:32]**，DW2 = **Address[31:2] + AT**，DW3 = **Requester ID + Tag + Last/1st DW BE**（其余同上）。

**完成头（Cpl / CplD，固定 3 DW）**

| DW  | 字段                     |  位宽  | 说明                                                |
| :-: | :--------------------- | :--: | :------------------------------------------------ |
| DW1 | **Completer ID[15:0]** |  16  | 完成方的 B:D:F（Switch 转发时会改写）                         |
| DW1 | **Cpl Status[2:0]**    |  3   | `SCC`(成功) / `UR`(不支持) / `CRS`(重试) / `CA`(被访问目标错误) |
| DW1 | **BCM / R**            |  1   | Byte Count Modified（拆包标记）                         |
| DW1 | **Byte Count[11:0]**   |  12  | 剩余待传字节数（≤ 4096 B）                                 |
| DW2 | **Requester ID[15:0]** |  16  | 回给谁（原请求 RID）                                      |
| DW2 | **Tag[7:0]**           | 8/10 | 原请求 Tag —— **RID + Tag 唯一确定一个未完成事务**              |
| DW2 | **Lower Address[6:0]** |  7   | 首字节在双字内的偏移（决定 1st DW BE）                          |

**消息头（Msg / MsgD）**：Fmt 标记是否带数据，Type 内含 **rrr 路由字段**（RC、广播、按 RID 上行/下行、按地址 GC/本地），DW1/头字节携带 **Message Code[7:0]**（如 `Assert_INTA#`、`Error_Poisoned`、`Latency_CMN`、Vendor Defined Type 0/1、PM 激活等）。

### 3.4 主要 TLP 类型对照

| Type                | 名称                      | 中文         |   类别    | 说明                                               |
| :------------------ | :---------------------- | :--------- | :-----: | :----------------------------------------------- |
| **MRd**             | Memory Read Request     | 内存读请求      | **NP**  | 32/64 位地址（Fmt 区分）；受 MRRS 限制；返回被切成 ≤ MPS 的多个 CplD |
| **MWr**             | Memory Write Request    | 内存写请求      |  **P**  | 带数据；最常用的高带宽事务（DMA 写内存）                           |
| **IORd / IOWr**     | I/O Read / Write        | IO 空间读写    | NP / P  | **兼容性保留**，现代设备基本不用（PCIe 4.0 起可完全不支持）             |
| **CfgRd0 / CfgWr0** | Type 0 Config           | 配置读写（本设备）  | NP / P  | 目标 = Endpoint/Switch 自身配置空间                      |
| **CfgRd1 / CfgWr1** | Type 1 Config           | 配置读写（转发）   | NP / P  | 由桥按 Bus 号路由，继续下行；跨桥时按规则降级为 Type0                 |
| **Cpl**             | Completion without Data | 无数据完成      | **Cpl** | 如 IOWr/CfgWr 的确认，或错误完成（UR/CA）                    |
| **CplD**            | Completion with Data    | 带数据完成      | **Cpl** | MRd/CfgRd 的数据载体                                  |
| **CplLk / CplDLk**  | Locked Completion       | 锁定完成       |   Cpl   | 配合传统原子锁序列（#REQ/#GNT），新设计少用                       |
| **Msg / MsgD**      | Message Request         | 消息（带/不带数据） |  **P**  | 中断 INTx 断言/撤销、错误上报、Vendor 定义、PM 握手等              |
| **AtomicOp**        | FetchAdd / Swap / CAS   | 原子操作       |    P    | 4.0+ 引入，在 RC/目标端原子更新                             |
| **DMWr**            | Deferrable Memory Write | 可延迟写       |   NP    | 5.0+ 引入，配合 vPMEM/持久内存语义                          |

### 3.5 DLLP（Data Link Layer Packet）

固定 **64 bit = 8 字节**，由 **6 个字段**（1 个类型 + 4 个信息字节 + 1 个 CRC）组成，**只在两段 Port 之间有意义，永不被 Switch 转发**。

```text
 ┌────────────────┬──────────┬──────────┬──────────┬──────────┬────────────────┐
 │ DLLP Type[15:0]│ Info B3  │ Info B2  │ Info B1  │ Info B0  │  CRC-16[15:0]  │
 │   16 bit       │   8 bit  │   8 bit  │   8 bit  │   8 bit  │    16 bit      │
 └────────────────┴──────────┴──────────┴──────────┴──────────┴────────────────┘
        合计 16 + 32 + 16 = 64 bit；4 字节信息区中含 **SeqNum[11:0]** / VC / Scale / Credit 计数
```

| 组           | DLLP                                                                               | 作用                                                                      |
| :---------- | :--------------------------------------------------------------------------------- | :---------------------------------------------------------------------- |
| **ACK/NAK** | **Ack** / **Nak**                                                                  | 携带被确认 TLP 的 **12 bit SeqNum**；Ack = 累积确认（该序号及之前全部正确），Nak = 请求从该序号起重传    |
| **流控**      | **InitFC1(P/NP/Cpl)**、**InitFC2(P/NP/Cpl)**、**UpdateFC(P/NP/Cpl)**                 | 初始化 Credit（两轮握手确认一致性）与周期刷新 Credit；含 VC 号、Hdr/Data 计数、**Scale（1/2/4/8）** |
| **电源管理**    | **PM_Enter_L1**、**PM_Enter_L23**、**PM_Active_State_Request_L1**、**PM_Request_Ack** | ASPM 进入 L1/L23 的请求与 ACK；L1 子状态相关                                        |
| **链路管理**    | Link Width Management、Lane Usage Management、Silent Training 等                      | 链路宽度重配、Lane 使用变更（Retimer/SRIS 相关）                                       |
| **厂商扩展**    | Vendor Specific Type 1 / Type 2                                                    | 厂商自定义诊断、私有链路特性                                                          |
| **调度**      | —                                                                                  | DLLP 与 TLP 竞争发送时，**控制类 DLLP 优先级高于 TLP**（重传/Ack 不能被数据饿死）                 |

### 3.6 Ordered Set（物理层训练/维护包）

| OS             | 长度（Gen1/2 → Gen3+ → Gen5） | 出现时机                                                   | 作用                                                                   |
| :------------- | :------------------------ | :----------------------------------------------------- | :------------------------------------------------------------------- |
| **TS1**        | 4 → 16 符号                 | Detect / Polling / Configuration / Recovery / Loopback | 携带链路号、Lane 号、速率能力、标志位（Auto-Change、Hot_Setup、Disconnected 等）          |
| **TS2**        | 4 → 16 符号                 | Polling.Configuration / Configuration / Recovery       | 与 TS1 结构类似，表示"已同意并锁定"，用于确认协商结果                                       |
| **FTS**        | 1~4 符号                    | **L0s → L0** 快速退出                                      | Fast Training Sequence，接收端据此快速重新锁定 CDR，无需完整重训                        |
| **SKP**        | 4 → 8/16 符号（按代次/时钟模型不同）   | 周期插入（Gen1/2 每 1180±2 个符号一次量级）                          | **时钟补偿**：吸收两端 ±300 ppm 频差，见下                                         |
| **SDS**        | 16 符号                     | Configuration.Idle / Recovery.Lock 之后                  | Start of Data Stream：宣告数据块开始，完成 128b/130b block 对齐与 Lane 去偏斜（deskew） |
| **EIOS**       | 4 符号                      | 进入 L0s/L1/L2 前                                         | Electrical Idle **Entry**：请求进入电气静默（省电）                               |
| **EIEOS**      | 16 符号                     | 高速切换、退出电气静默后                                           | Electrical Idle **Exit** OS：让对端重新获得 bit lock，之后接 FTS/TS              |
| **Compliance** | 16 符号                     | Polling.Compliance                                     | 一致性测试模式（认证/仪器测量）                                                     |

**SKP 时钟补偿原理（必考）**

两端各自用独立（或扩频）100 MHz 参考时钟，存在 ±300 ppm 偏差：1 GHz 下即每秒漂移约 100 万 bit。物理层因此设计**弹性缓冲（Elastic / SKP Buffer）**：

```text
发送端（按固定周期插入）    … D1 D2 D3 │ SKP SKP SKP │ D4 D5 D6 …
接收端弹性队列（读指针按恢复时钟走）
    · 平均水位偏低 → 原样收下 3 个 SKP，只删除多余 SKP（吸收"对端比本机慢"）
    · 平均水位偏高 → 增补 SKP（伪造 SKP）以延缓读出，保持 FIFIO 不溢出（吸收"对端比本机快"）
    · 删除/插入必须整组对齐，且保证 SKP 段前后的数据相位连续 → 数据本身永不受影响
```

- SKP Ordered Set 是**唯一允许被接收端增/删**的 Ordered Set，因此它不能承载任何端到端语义。
- 弹性缓冲水位超限 → **SKP Underflow/Overflow**，链路降级/重训练（Recovery），是高速链路 SI 裕量不足的经典症状。
- **SRIS**（独立扩频时钟）频差加倍，SKP 插入密度需更高，弹性缓冲要更深；**SRNS**（共时钟）余量最松。

---

## 🔄 4. LTSSM：链路训练状态机

每个 Port 的物理层都跑一份 LTSSM，双向协同。核心路径 **Detect → Polling → Configuration → L0**。

| 状态                | 主要子状态                                                                                | 作用与退出条件                                                                                                                                 |
| :---------------- | :----------------------------------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| **Detect**        | Quiet / Active / Chirp                                                               | 发送端对 Lane 做**接收器检测**（阻抗充放电判断对端是否存在）。任一 Lane 检测到对端 → **Polling**；未检测到 → 回 Quiet 重试。约 12 ms 量级                                            |
| **Polling**       | Active / Compliance / Configuration                                                  | 双方互发 **TS1/TS2**，完成 bit lock、符号/块对齐、Lane→Polarity 校正与极性反转锁定；满足条件（如收到 8 个连续匹配 TS2）→ Configuration；一致性测试模式 → Compliance                   |
| **Configuration** | Linkwidth.Start / Linkwidth.Accept / Lanenum.Wait / Lanenum.Accept / Complete / Idle | **协商链路宽度与 Lane 编号**：确定 x1~x32 中的实际宽度、Lane 顺序（含 Lane Reversal）、速率（Master/Slave）、是否启用 SRIS；配置完成发 **SDS** 并进入 **L0**（Gen3+ 通常先跑 Gen1 再切高速） |
| **L0**            | —                                                                                    | **正常工作态**：收发 TLP/DLLP/OS，做流控与重传                                                                                                         |
| **L0s**           | Rx / Tx 子态                                                                           | 单 Lane 浅睡眠，靠 **FTS** 微秒级快速唤醒；进入时先发 EIOS，不通知对端方向                                                                                         |
| **Recovery**      | Lock / Equalization / RcvrCfg / Idle                                                 | **重训练**：速率切换（如 Gen1↔Gen3↔Gen5）、均衡协商（Phase 0~3，含 preset/系数请求）、Lane 去偏斜、失锁/Framing 错误恢复；成功回 **L0**，失败退到 Gen1 或转 Configuration/Detect      |
| **L1**            | L1 / L1.1 / L1.2（子状态）                                                                | 深低功耗：电气静默、参考时钟可关（L1.2 只留时钟检测），靠 **PM DLLP** 握手进入，退出走 Recovery；唤醒延迟高于 L0s                                                                |
| **L2**            | —                                                                                    | 更低功耗（主电源可撤），仅保留辅助电源与唤醒（WAKE#/Beacon）；退出需重新走 Detect→… 完整训练                                                                               |
| **Hot Reset**     | —                                                                                    | 上游在 Recovery/Configuration 中持续发送 **TS1 且 Hot_Setup=1**（或直接拉低 PERST#），下游跟随复位，最终回到 Detect                                                 |
| **Loopback**      | Entry / Active                                                                       | 测试态：进入方回环收到的比特流，用于误码/导通测试（`Loopback` 位）                                                                                                 |
| **Disabled**      | —                                                                                    | 收到 **Disable** 链路（如 TS1 中 Disable 位、软件 Link Disable）→ 关断发送；一般需复位/重新上电才恢复                                                                |

```text
        上电 / PERST# 释放
                │
                ▼
   ┌────────┐  检测到对端接收器  ┌─────────┐   锁定 + TS2  ┌──────────────┐
   │ Detect ├─────────────────►│ Polling ├──────────────►│Configuration │
   └────────┘                  └─────────┘               └──────┬───────┘
        ▲   未检测到→重试(≈12 ms)      │ 一致性问题                │ 宽度/Lane/速率确定
        │                            ▼                          │ SDS
        │                    Polling.Compliance(测试)            ▼
        │              ◄──────────────────────────────────────► L0 ◄──────┐
        │                                                       │  ▲      │
        │            速率/均衡变更、失锁、Framing 错、SKP 溢出     ▼  │成功   │
        │                                    ┌─────────── Recovery ──┘      │
        │                                    │ 失败：降速/回 Detect         │
        │  Hot Reset（TS1 Hot_Setup / PERST#）│                             │
        └───────────────────────────────────►┘      ASPM：L0s / L1 / L2 ◄───┘
```

> [!TIP] 调试抓手
> - "跑不到 Gen3/Gen5、只停在 Gen1" 或 "频繁 Recovery" 99% 是 **SI/均衡问题**：看 Recovery.Equalization 用的 preset、信道损耗与眼图余量、SKP 溢出计数。
> - "只有 x4/x8、达不到 x8/x16" → Configuration 阶段的 **Lane 位图**问题：查某条 Lane 的导通/AC 耦合/skew。
> - 读 **Lane Status / Link Status 寄存器**（`Current Link Speed`、`Negotiated Link Width`、`Link Equalization Request`）是最快的判定路径。

---

## 🛡️ 5. 关键机制：流控、重传与数据完整性

### 5.1 基于 Credit 的流控（FC）

- 三类流量 × 两类资源 = **6 个 Credit 计数器**：`Posted(P)` / `Non-posted(NP)` / `Completion(Cpl)`，每类分别计 **Header Credit** 与 **Data Credit**；后续规范引入 Shared Credit 进一步细化。
- 规则：**发一个 TLP 就要消耗对应 Credit；接收端处理并腾空缓冲后通过 UpdateFC DLLP 归还**。发送端 Credit 不足即停发该类别——**背压，绝不丢包，也绝不过载**。
- 初始化：**InitFC1 → InitFC2 → UpdateFC** 三轮握手，保证双方对初始值认知一致（防止误信造成死锁）；链路必须处于 L0 且 Credit 一致才允许发 TLP。
- 刷新节奏：UpdateFC 有最大延迟约束（规范给出百微秒量级上限，实现通常在几十 μs 内），因此**空闲链路的 Credit 可能滞后**。
- 关键设计点：**"数据类 Credit（尤其 Cpl Data）不够"是完成风暴/读带宽上不去的头号原因**。Credit 总量要能覆盖往返时延：

$$
BW_{\max} \le \frac{\text{Credit 总数据量}}{t_{\text{round-trip}}}
$$

| 常见配置项 | 寄存器 / 概念 | 影响 |
| :--- | :--- | :--- |
| 最大净载荷 | Device Control.**MPS**（128/256/512/1024/2048/4096 B） | 越大头开销占比越低；路径上所有设备取**最小值**，配置不匹配会造成 Malformed TLP |
| 最大读请求 | Device Control.**MRRS** | 决定读切包粒度 |
| 并发读 | **Tag** 数量（8/10 bit） | 上限 = outstanding 读请求数，与 Credit 共同决定实际读带宽 |
| 虚拟通道 | VC / TC 仲裁 | 用 TC 区分实时（显示/音频）与尽力而为流量 |

**排序规则（Ordering Rules）要点**（避免死锁 + 保持生产者/消费者语义）：

| 规则 | 是否允许"后面包超越前面的包" | 原因 |
| :--- | :--- | :--- |
| Posted 越过 Non-posted / Completion | **允许** | 写通道不能被慢速读阻塞，否则缓冲打满死锁 |
| Completion 越过 Posted | **允许** | 完成必须能穿过写流，否则 RC/桥写排空与读返回互相卡死 |
| **Non-posted 越过 Posted** | **禁止** | 保证"先写后读"的可见性（读不能回到已发出写之前） |
| Posted 越过 Posted / Completion 越过 Completion | **默认禁止**，置 **Attr[1]=RO** 后允许 | 放宽以消除大规模并行 DMA 的队首阻塞（HOL） |

> [!IMPORTANT] 生产者/消费者陷阱
> 设备 DMA 先写数据、再写"完成标志"到同一目标（如主机内存 ring 的描述符），必须保证**数据写先于标志写可见**：要么两笔都 `RO=0`，要么在标志前插入读回（`MRd` 是 Non-posted，天然不能越过前面的写）。这是 [[DMA、Buffer 与流量控制]] 中"写顺序"问题的 PCIe 根源。

### 5.2 Ack / Nak 重传与隐藏重试缓冲

```text
发送端                                     接收端
  TLP(Seq=101) ──LCRC 错──► 丢弃，回 Nak(101)   ← 检 LCRC/Framing 失败
  Retry Buffer 保存 101~105
  收到 Nak(101) → 从 101 起按序重发 101,102,…,105（TLP 序号不变）
                                    接收端按 SeqNum 丢弃重复（duplicate discard）
  收到 Ack(105) → 释放 101~105 的重试缓冲空间
```

- **累积确认**：Ack(N) 表示"N 及之前全部正确接收"，一个 DLLP 即可确认多个 TLP。
- **重传窗口**：由 **Ack Latency Timer（接收端允许延迟确认的时间）** + **链路传播时延** 决定，Retry Buffer 必须**足够深**以覆盖"已发出但未确认"的全部 TLP——这部分容量对软件不可见，即**隐藏重试缓冲（Hidden Retry Buffer）**；它同时决定 Gen4+ **Flow Control Credit Scaling** 与高速长链路下的可用带宽。
- **REPLAY_TIMER**：超时未收到 Ack（典型几十微秒量级）即自动整窗重传；重传次数超限（`REPLAY_TIMER_TIMEOUT` 计数）→ 上报错误并进入 **Recovery / Disable Link**。
- **序号回绕**：12 bit 循环（1~4095，`0` 为特殊/无效），窗口必须小于 4096/2 才不歧义，规范通过限制未确认 TLP 数量保证这一点。

### 5.3 完整性与错误处理层次

| 机制 | 覆盖范围 | 由谁负责 | 能否纠错 | 开销 |
| :--- | :--- | :--- | :--- | :--: |
| **编码规则检查**（8b/10b 非法码/disparity；128b/130b block type） | 链路本地 | PL | 仅检错 | 编码内含 |
| **DLLP CRC-16** | 链路本地 | DL | 检错，DLLP 直接丢弃（Ack 丢了靠超时重发） | 16 bit |
| **LCRC（32 bit CRC）** | 链路本地（每 hop 重算） | DL | 检错 → **Nak 重传** | 4 B |
| **ECRC（可选，TD=1）** | **端到端** | TL | 检错，跨过 Switch 仍能发现数据损坏 | 4 B |
| **Poisoned（EP 位）** | 端到端 | TL | 标记"这段数据已不可信"，交由软件/AER 处理 | 1 bit |
| **Sequence Number + Ack/Nak + Retry Buffer** | 链路本地 | DL | 保证**不丢、不重、有序** | 2 B + 隐藏缓冲 |
| **FEC（Gen6）** | 链路本地 | PL | **前向纠错**（PAM-4 误码率过高，必须先纠再重传） | FLIT 内字节 |
| **DLP / Poisoned / Malformed 错误 + AER 扩展能力** | 上报 | TL/DL/PL | 记录 + 中断/系统处理 | 寄存器 |

---

## 🧩 6. 配置空间、BAR 与总线枚举

### 6.1 拓扑与地址

- 每个功能（Function）有 **256 B 标准配置空间**（PCI 兼容）+ **PCIe 扩展配置空间到 4 KB**（偏移 ≥ 0x100）。
- **Requester/Completer ID = 16 bit**：`Bus[15:8] : Device[7:3] : Function[2:0]` → 每条总线最多 **32 设备 × 8 功能 = 256 个 Function**。
- 软件通过 **ECAM**（Memory-Mapped Config）访问：`base + (Bus<<20) + (Dev<<15) + (Func<<12) + offset`，因此一次配置读写在链路上就是一笔 `CfgRd0/1`、`CfgWr0/1` TLP。

### 6.2 Type 0 与 Type 1 头（关键偏移对照）

| 偏移 | Type 0（Endpoint / 非桥设备） | Type 1（PCI-PCI Bridge / Switch Port） |
| :--: | :--- | :--- |
| 0x00 / 0x02 | Vendor ID / Device ID | 同 |
| 0x04 / 0x06 | **Command**（Bus Master / SERR / INTx Disable / IO·Mem Enable）/ **Status** | 同 |
| 0x08 / 0x09 | Rev ID / **Class Code**（0x060400 = Bridge → Type 1） | 同 |
| 0x0C / 0x0D | Cache Line Size / Latency Timer（PCI 兼容，多数无效） | 同 |
| 0x10~0x24 | **BAR0~BAR5**（最多 6 个 32 位或 3 个 64 位） | BAR0、BAR1（均为内存窗口，非 BAR 语义） |
| 0x28 | — | **BIOS/Card Bus 地址寄存器** |
| 0x2C / 0x2E | Subsystem Vendor ID / ID | 同（桥可选） |
| 0x30 | Expansion ROM BAR | Expansion ROM 基址 |
| 0x34 | **Capabilities Pointer**（PCI 能力链表，含 PCI Express Capability） | 同 |
| 0x38 | — | 保留 / **Bridge Control** |
| 0x3C / 0x3D | Interrupt Line / **Interrupt PIN** | 同（桥通常 INTA~D 直通） |
| 0x18~0x1D | — | **Primary / Secondary / Subordinate Bus Number**、秒级计时 |
| 0x1C~0x28 | — | **Memory Base/Limit、Prefetchable Base/Limit（含 32 位 upper 段）** |
| 0x30~0x34 | — | **I/O Base / I/O Limit** |
| 0x3A~0x3B | — | Bridge Control（Secondary Bus Reset 等） |
| 0x100+ | PCIe 扩展能力：AER（0x01）、VC、Device3、**ACSL**、SR-IOV、PTM、DF、VF 等 | 同 |

> PCIe Capability（通常在 0x40~0x80 之间的某处，由能力链索引）里最常用的是 **Link Capabilities / Link Status / Device Control**：`Max Link Speed`、`Max Link Width`、`Current Link Speed`、`Negotiated Link Width`、`Data Link Layer Link Active`、`MPS/MRRS`、ASPM 使能位。

### 6.3 BAR 工作原理（写全 1 → 读回 → 算大小）

1. 软件向 `BARn` 写 **0xFFFFFFFF**（全 1）。
2. 设备把不实现的**低位强制为 0**（体现该 BAR 的最小对齐粒度），并保留**类型标志位**。
3. 软件读回值：`低位 0 的个数` 决定大小；`bit[2:1]`（内存型）区分 32 位 / 64 位 / prefetchable；`bit0=1` 表示 IO 空间 BAR。
4. 软件写回**分配到的基地址**（高位），低位保留类型位，即完成映射。

$$
\text{BAR 大小} = \overline{(\text{读回值} \,\&\, \texttt{0xFFFFFFF0})} + 1
$$

**示例**：设备实现 16 KB 的 32 位内存 BAR。

```text
写 0xFFFFFFFF → 读回 0xFFFFC001
  bit0 = 1        → Memory BAR（PCIe 只支持 memory 语义）
  bits[2:1] = 00  → 32-bit BAR（若 10 = 64-bit，需占用下一个 BAR 拼成 64 位）
  bit3 = 0        → 非 prefetchable
  掩掉低 4 位：0xFFFFC001 & 0xFFFFFFF0 = 0xFFFFC000
  大小 = ~0xFFFFC000 + 1 = 0x00004000 = 16 KB
  → 若需 64-bit 形式：读回高 32 位 BAR 为 0xFFFFFFFF（大小由低 BAR 的掩码决定）
```

> [!WARNING] 枚举/重映射常见错误
> - **64 位 BAR 占两个连续槽**：分配时不能只搬动一个，也不能把 IO BAR 与它拆开。
> - 桥的 Memory/IO 窗口按 **1 MB（Mem）/ 4 KB（IO）** 粒度，向下取整分配；先自底向上统计子树需求，再自顶向下填窗口，否则"窗口装不下"导致 BAR 分配失败。
> - `BAR size = 0` 表示**该 BAR 未实现**（读回 0）；驱动要能容忍。
> - 重定位 BAR 前必须**关闭 Memory/IO Enable 并完成在途事务**（`Command` 寄存器位清零 + 对该设备做一次配置读确保排空）。

### 6.4 总线枚举流程简述

```text
① RC 复位后只有 Bus 0；从 Bus 0 / Dev 0~31 / Func 0~7 逐个 CfgRd0 Vendor ID
     │ 读回 0xFFFF → 无设备（Master Abort）；读回真实 VID → 存在
② 读 Header Type (0x0E)：bit7=1 → 多功能；Type=01b → 是桥
③ 遇到桥（Switch Downstream/Root Port）：
     临时把 Primary=当前 Bus，Secondary=Subordinate=一个尚未使用的最大 Bus 号（如 255）
④ 以"深度优先"下行遍历：桥下的 CfgRd1 会被继续转发 → 发现子总线上的设备
⑤ 回溯时给桥定 Secondary/Subordinate（自底向上确定子树最大 Bus 号）→ 形成 Bus 区间
⑥ 对每个 Endpoint：读 BAR 尺寸 → 在父桥的窗口内分配基地址 → 写回 BAR
⑦ 遍历能力链：置 MPS/MRRS、使能 Bus Master、配置 ASPM/L1 子状态、
   配置 MSI/MSI-X（地址=中断重映射表项，Data=向量号）、使能 AER
```

- 配置事务的**广播与定向**：`CfgRd1/CfgWr1` 在未指定目标 Bus 时可广播（用于发现 Dev 0），指定 Bus 后由桥逐级转为 Type 1/Type 0 直到目标 Port。
- 同一时刻 **每个总线段只允许一条在途配置链**，配置空间访问是"慢路径"，禁止在 ISR 中做长时间配置轮询。
- `Class Code` + `Vendor/Device ID` + **DSN（Device Serial Number 能力）** 用于驱动绑定与拓扑唯一标识。

---

## 🔔 7. 中断：INTx / MSI / MSI-X

| 特性 | **INTx（虚拟线）** | **MSI** | **MSI-X** |
| :--- | :--- | :--- | :--- |
| 物理信号 | **无**（用 `Msg` TLP 表达 Assert_INTA#…Assert_INTD# 及撤销消息） | 无 | 无 |
| 本质 | 电平语义，Posted **Msg** | 一笔对主机内存的 **Posted MWr**（地址=消息地址，数据=消息数据） | 多笔独立 MWr，向量表在设备 BAR 中 |
| 向量数 | 4（A/B/C/D），且常需共享 | **1/2/4/8/16/32**（2 的幂，MME 选择，须连续分配） | **最多 2048**，任意分配，不要求 2 的幂 |
| 每向量地址 | 共享 | 全设备共用一个地址+数据（仅低位区分向量） | **每向量独立 Address + Data + Mask**（表项 16 B） |
| 屏蔽 | 设备级 `INTx Disable`（Command[10]） | 可选 per-vector mask（4.0+）/ 向量控制寄存器 | **每向量独立 Mask 位**，另有 **PBA**（Pending Bit Array） |
| 撤销 | 需写设备状态寄存器 + 引擎撤销消息 | 边沿语义：写清设备内状态即可 | 同 |
| 适用 | 兼容旧系统/共享中断 | 单队列设备 | **NVMe/网卡多队列、SR-IOV VF**，一队列一中断，天然避免假共享 |
| 主要坑 | 共享 + 电平 → 中断风暴、需要轮询所有可能源 | 数据被广播/排序问题；某些平台要求写后 flush | Table 位于 BAR 中，**必须正确映射并按 4 KB 隔离安全边界**（否则 guest 可伪造中断） |

> [!NOTE] MSI 与"写排序"的关系
> MSI 就是一次普通 Posted MWr，因此**遵循 §5.1 的排序规则**：中断写不会越过它前面的设备状态写/数据 DMA 写（Non-posted 不能越过 Posted，Posted 同向默认有序，除非 RO）。这也是"先清中断状态再返回 IRQ_HANDLED"这类驱动约定在 PCIe 上依然成立的原因。

---

## 🔋 8. 电源管理：D 状态与 L 状态

**设备侧 D 状态（Function 粒度，配置空间 Power Management Capability）**

| 状态 | 含义 | 约束 |
| :--- | :--- | :--- |
| **D0** | 全速工作（`D0unpop`/`D0cold`/`D0hot` 细分上电过程） | 完整功能 |
| **D1** | 低功耗，功能部分保留，可无损恢复 | 实际支持很少 |
| **D2** | 更低功耗，可能丢失内部状态，需要重配置 | 唤醒延迟大 |
| **D3hot** | 主电源在、功能基本关断；配置空间可访问（写全 1 读回不再可靠） | 必须写 `PME_En`/转换到 D3 前保存上下文 |
| **D3cold** | **主电源断开**，只有辅助电源或完全断电 | 只能靠 PME/唤醒事件重新上电，恢复后必须重走完整初始化/重枚举 |

**链路侧 L 状态（Link 粒度，ASPM）**

| 状态 | 进入方式 | 退出代价（数量级） | 说明 |
| :--- | :--- | :--- | :--- |
| **L0 / L0p** | 正常传输（L0p 为 Gen6 动态宽度） | — | 全速 |
| **L0s** | 空闲自动进入（每 Lane 独立，发 EIOS） | 纳秒~微秒（FTS 快速重锁） | 只省电气/时钟功耗，**不关参考时钟** |
| **L1** | `PM_Enter_L1` / `PM_Active_State_Request_L1` → `PM_Request_Ack` DLLP 握手 | 几微秒~几十微秒 | 电气静默，参考时钟保留 → **L1.1**；关参考时钟 → **L1.2**（依赖 **CLKREQ#** 拉低） |
| **L1.1 / L1.2（L1 PM Substates）** | 在 L1 内配合 LTR + Clock PM | 更大（唤醒需等参考时钟稳定） | 笔记本/移动端主要省电手段，需 OS+平台+设备三方一致 |
| **L2** | `PM_Enter_L23`，链路彻底关断，只留辅助电源 | 完整重训练（回 Detect） | 唤醒靠 WAKE#/Beacon，D3cold 通常对应 L2/L3 |
| **L3** | 电源完全移除 | — | 无任何上下文 |

> [!TIP] 省电与延迟的取舍
> ASPM 打开后 **L1 退出延迟会计入设备树路径上所有桥的累积值**（规范用 `L1 Exit Latency` / LTR 机制表达）。实时设备（NVMe 队列、GPU、音频）常关闭 ASPM 或只开 L0s；用 `lspci -vvv` 查 `LnkCap: ASPM … Exit Latency` 与 `DevCtl: ASPM Disabled` 是标准核对手段。

---

## 🖼️ 9. PCIe 帧结构与链路训练时序图

![[PCLe.excalidraw]]

- **左画布**：TLP 4DW 头字段逐位排布（DW0~DW3）、线路上完整 TLP 封装（STP / SeqNum / Header / Data / ECRC / LCRC）、以及 **DLLP 6 字段**（16b Type + 4×8b Info + 16b CRC-16）与 SeqNum/LCRC 对应关系。
- **右画布**：Host(Root Port) ↔ Device(Endpoint) 的 LTSSM 消息时序：上电与接收器检测 → TS1/TS2 交换 → 链路宽度/Lane 号协商 → 速率切换与均衡 → SDS 对齐 → 进入 L0，并标注最终协商结果（Lane 数 / 速率）。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] PCIe 链路设计与调试必过
> - [ ] **拓扑与 Lane**：x1/x4/x8/x16 与目标设备一致；差分对**交叉连接**（TX±→RX±）、极性可反转、支持 Lane Reversal
> - [ ] **电气**：AC 耦合电容值与位置正确、差分阻抗 85/100 Ω、对内/对间 skew 在预算内、参考时钟 100 MHz 与 ±300 ppm（SRNS/SRIS 选择明确）
> - [ ] **速率与均衡**：实测 `Current Link Speed` 达设计代次（注意 64 GT/s 是 **Gen6**）；Gen3+ 均衡 Phase 0~3 与 preset 已调；`Recovery` 进入计数为 0
> - [ ] **链路宽度**：`Negotiated Link Width` 等于设计值；无单 Lane 掉线
> - [ ] **LTSSM**：能稳定停在 **L0**；Polling/Configuration 无反复；Disable/Hot Reset 非预期时先查 PERST# 时序
> - [ ] **编码/SKP**：Gen3+ 对齐依赖 block/SDS（无 COMMA）；SKP 溢出/弹性缓冲计数为 0
> - [ ] **MPS / MRRS**：路径上**所有**设备 MPS 取最小值且无冲突；MRRS 与读带宽目标匹配
> - [ ] **流控 Credit**：InitFC1/InitFC2/UpdateFC 正常；Credit 总量 ≥ 往返时延 × 目标带宽；无 `Flow Control Protocol Error`
> - [ ] **重传与完整性**：NAK/Replay 计数低；`REPLAY_TIMER_TIMEOUT` 计数为 0；ECRC/Poisoned 策略明确
> - [ ] **排序**：多队列 DMA + 完成标志是否满足"数据先于标志可见"（RO 位设置正确）
> - [ ] **配置空间**：BAR 写全 1 读回掩码算尺寸正确、64 位 BAR 占两个槽；桥的 Memory/Prefetch/IO 窗口覆盖所有子树
> - [ ] **枚举**：VID/DID 可读、`Header Type` 判定桥、Bus 号区间（Primary/Secondary/Subordinate）无重叠
> - [ ] **中断**：MSI/MSI-X 使能且向量数足够；INTx 场景 `INTx Disable` 与状态清除顺序正确
> - [ ] **电源**：ASPM/L1 子状态与 `L1 Exit Latency` 满足实时性要求；D3cold 后有完整重初始化路径
> - [ ] **错误上报**：AER（Correctable / Uncorrectable）已配置并可观测；`Device Status` 的 Non-Fatal/Fatal 被处理

---

## 🔗 思考与延伸索引

- [[UART]] — 同为"分层与电气解耦"的典型：UART 靠约定波特率自同步，PCIe 靠 **8 GHz 以上 CDR + SKP/Ordered Set** 自同步；对比两者"帧 vs 包"的差错控制差异
- [[SPI]] — 同步、共时钟、主从一一对应；PCIe 把"时钟"藏进串行流（嵌入时钟 + 加扰），并交换化以扩展多设备
- [[I2C]] — 多主、共享、开漏仲裁、7/10 bit 地址；与 PCIe 的"点对点 + 交换机 + 地址路由"形成鲜明对照
- [[DMA、Buffer 与流量控制]] — PCIe 的 **Credit 流控 / 弹性缓冲 / 重试缓冲** 是"用缓冲换不丢包"这一通用思想的协议化实现
- [[USB]] — 同样是串行分组 + 训练（USB 的速率/角色协商）+ 分层，但在枚举模型（描述符 vs 配置空间）、流控（NAK/时间片 vs Credit）上完全不同

---

## 📚 参考资料

1. **PCI-SIG** — *PCI Express® Base Specification Revision 5.0 / 4.0*（分层、TLP/DLLP/OS 格式、LTSSM、流控与排序规则、配置空间与电源状态的权威定义）.
2. **PCI-SIG** — *PCI Express Base Specification Revision 6.x*（PAM-4、1b/1b + FLIT、FEC、L0p 动态宽度）.
3. **PCI-SIG** — *PCI Express Electromechanical (CEM) Specification*（连接器、信道损耗预算、AC 耦合与参考时钟拓扑）.
4. **MindShare, Inc.** — *PCI Express® Architecture 3.0*（Ed Mindess 等，事务/链路/物理层与 TLP/DLLP 讲解的经典参考）.
5. **MindShare, Inc.** — *PCI Express Technology 3.0*（枚举、BAR、MSI/MSI-X、AER 与软件视角）.
6. **Mike Jackson & Ravi Budruk** — *Desktop PCI Express: Technology and Applications*, Intel Press.
7. **PCI Industrial Architects Mechanism (PIAM) / DMTF** 相关文档与 **Linux ` Documentation/PCI/` 、`lspci(8)`/`setpci(8)`（pciutils）** — 枚举、能力结构与调试实操参考.
8. **Texas Instruments / Parade / Diodes** 等厂商 **PCIe Redriver / Retimer / Switch App Note**（眼图、均衡 preset、SKP 与弹性缓冲的实战约束）.
