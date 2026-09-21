---
title: DMA、Buffer 与流量控制（背压）完整技术手册
tags:
  - Embedded
  - DMA
  - Buffer
  - Backpressure
  - UART
  - SPI
  - I2C
  - Architecture
status: Completed
created: 2026-09-20
---

# 📝 DMA、Buffer 与流量控制（背压）完整技术手册

> [!INFO] 笔记概览
> - **核心命题**：DMA 解决"搬运效率"，Buffer 解决"短期速率不匹配"，背压解决"控制生产速度"——三者职责不可混淆。
> - **内容范围**：DMA 职责边界 · TC 语义 · 生产者/消费者模型 · RX 溢出根因 · Circular/Ring/Double Buffer · 高低水位背压 · UART/SPI/I²C 流控能力对比
> - **推荐路径**：§1 DMA 本质 → §2 TC 语义 → §3 生产消费模型 → §4 溢出根因 → §5~6 Buffer 管理 → §7 背压 → §8 外设对比 → §9 统一模型

---

## 🚚 1. DMA 的本质与职责边界

DMA（Direct Memory Access）本质上只做一件事：

> **在外设与内存之间自动搬运数据，减少 CPU 参与数据搬运的开销。**

```text
外设 ──► DMA ──► RAM      (RX：接收方向)
RAM  ──► DMA ──► 外设      (TX：发送方向)
```

### 1.1 DMA 不负责什么

DMA 是"搬运工"而非"理解者"，它**不涉及**任何协议语义：

| DMA 负责 | DMA 不负责 |
| :------- | :--------- |
| 按配置在外设↔内存间搬运字节 | 协议解析 |
| 维护传输计数、触发 TC/HT 中断 | 数据包边界判断 |
| 循环/单次地址递增 | CRC / 帧格式解析 |
| —— | "这是不是完整的一帧" |

> [!NOTE] 一句话分工
> **DMA 负责"搬"，CPU / Parser 负责"理解"。** 判断帧边界必须依赖外设事件（UART IDLE、SPI CS、I²C STOP）或软件协议解析，绝不能指望 DMA TC。

---

## 🏁 2. DMA TC（Transfer Complete）的真实语义

DMA 的 `TC（Transfer Complete）` 仅表示：

> **本次配置的 DMA 传输数量已经搬完。**

```text
DMA 配置：Buffer = 1024 B, Transfer Count = 1024
       │
       ▼
   搬完 1024 B ──► TC 置位 ──► 触发中断
```

它**并不意味着**"整个通信协议事务结束了"。TC 的业务含义必须结合具体外设判断：

| 外设 | DMA TC 的字面含义 | 真正的事务边界由什么决定 |
| :--- | :---------------- | :----------------------- |
| **UART RX** | 配置数量的数据搬完 | IDLE 中断、报文长度、协议帧定界 |
| **UART TX** | 数据已从内存搬到外设 | TX Complete / `TC` / `BSY` 标志 |
| **SPI**     | 配置的数据搬完 | CS/NSS 电平、传输长度 |
| **I²C**     | 配置的数据搬完 | STOP 条件、地址阶段、读写长度 |

> [!CAUTION] 经典误区：DMA TC ≠ 物理发送完成
> UART TX 场景中，DMA TCIF 只代表数据写入了串口数据寄存器（TDR），移位寄存器里的最后字节可能尚未离开引脚。若据此立即拉低 RS-485 的 DE，会截断帧尾——详见 [[modbus]] §2 的"交接棒"时序。

---

## 🔄 3. 生产者 / 消费者模型：TX 与 RX 的方向辨析

理解 DMA 溢出问题，第一步永远是**确定数据方向**，进而识别谁是 Producer、谁是 Consumer。

### 3.1 TX 方向：MCU → 外部设备

以 LVGL 刷屏为例：

```text
LVGL ──► Display Buffer ──► SPI DMA ──► SPI ──► LCD
(Producer)                                (Consumer)
```

当 `LVGL 产生速度 > SPI 发送速度` 时，表现为 LVGL 排队等待，**刷新吞吐量被 SPI 带宽限制，FPS 上不去**。数据不会被无限"冲进 DMA"，因为可以用 DMA 完成回调形成同步：

```text
LVGL 提交 Buffer A ──► SPI DMA 发送 ──► TC ──► flush_ready ──► LVGL 可继续提交
```

> [!TIP] TX 天然自限流
> 发送侧的 Producer（CPU/LVGL）本身就是可控的，通过 `flush_ready` 回调即可把发送速度锁死在 DMA/SPI 能承受的范围内，因此 TX 方向极少溢出。

### 3.2 RX 方向：外部设备 → MCU

接收方向完全相反，Producer 在 MCU 之外、不可控：

```text
外部设备 ──► DMA ──► Buffer ──► CPU / Parser
(Producer)                       (Consumer)
```

当 `外部产生速度 > CPU 处理速度`，数据持续积累：

```text
外部设备  ████████████████████  (10 MB/s)
DMA       ████████████████████
CPU       ███████               (3 MB/s)
                └── 缺口不断拉大 → Buffer 越来越满
```

最终链路：`Buffer 满 → 新数据继续到达 → 覆盖 / FIFO Overflow → 丢数据`。

---

## 📥 4. RX 溢出的根因：为什么增大 Buffer 治标不治本

设外部设备生产速率 $R_p$、CPU 消费速率 $R_c$，当 $R_p > R_c$ 时，Buffer 的积累速率为：

$$
R_{\text{accum}} = R_p - R_c
$$

从空到溢出（容量 $C$）所需时间为：

$$
T_{\text{overflow}} = \frac{C}{R_p - R_c}
$$

> [!WARNING] 扩容只是把溢出时间往后推
> 以 $R_p = 10\ \text{MB/s}$、$R_c = 3\ \text{MB/s}$ 为例，$R_{\text{accum}} = 7\ \text{MB/s}$：
>
> | Buffer 容量 $C$ | 溢出时间 $T_{\text{overflow}}$ |
> | :-------------: | :----------------------------: |
> | 4 KB   | ≈ 0.57 ms |
> | 4 MB   | ≈ 0.57 s  |
> | 40 MB  | ≈ 5.7 s   |
>
> 只要 $R_p > R_c$ **长期成立**，无论 Buffer 多大最终都会满。

> **结论：扩大 Buffer 只能延缓溢出、吸收突发，不能解决长期吞吐量不匹配。** 真正的解法是背压（§7）或提高消费速度。

---

## 🔁 5. Circular DMA 与 Ring Buffer

### 5.1 Circular DMA（硬件机制）

普通数组即可配置为 Circular DMA：

```c
uint8_t rx_buf[4096];   /* 配置为 DMA Circular 模式 */
```

硬件行为——写到末尾后自动回到头部：

```text
0 ──────────────────────► 4095
▲                            │
└──────────── 自动回绕 ──────-┘
```

### 5.2 软件 Ring 管理

软件维护两个指针，配合硬件回绕形成环形队列：

```c
size_t dma_pos;    /* DMA 当前写位置（由 CNDTR 反推） */
size_t read_pos;   /* CPU 已消费到的位置 */
```

```text
        ┌──────────── Buffer (4096 B) ───────────┐
read_pos│   已消费   │   待消费新数据   │  空闲   │dma_pos
   ─────┴────────────┴──────────────────┴─────────┴─────►
              新数据长度 = (dma_pos - read_pos) mod 4096
```

其中 DMA 写位置可由剩余计数反推：$\text{dma\_pos} = \text{BufferSize} - \text{CNDTR}$。

> [!NOTE] 两个概念别混淆
> **Circular DMA 是硬件的循环写入机制；Ring Buffer 是软件对这块内存的管理方式。** 二者常配合使用，但不是同一层的东西。

> [!WARNING] Circular DMA ≠ 无限缓存
> Circular DMA 不增加 Buffer 容量。若 `DMA 写速度 > CPU 消费速度`，DMA 绕回后会**写到 CPU 还没消费的位置 → 覆盖旧数据 → 丢数据**。必须用高水位检测（§7）防止写指针追上读指针。

---

## 🧩 6. Normal DMA、Ping-Pong 与 Double Buffer

不使用 Circular 模式时，采用 Normal DMA + 分块处理：

```text
Buffer A: 0 ──► END ──► TC ──► 处理数据 ──► 重新配置 DMA
Buffer B: 0 ──► END ──► TC ──► ...
```

进一步用两块 Buffer 交替，形成 **Ping-Pong / Double Buffer**，让 DMA 与 CPU 并行：

```text
┌─ 时刻 1 ─┐   ┌─ 时刻 2 ─┐    ┌─ 时刻3 -─┐
│ DMA → B  │   │ DMA → A  │   │ DMA → B  │
│ CPU → A  │   │ CPU → B  │   │ CPU → A  │
└──────────┘   └──────────┘   └──────────┘
```

| 管理方式 | 特点 | 适用场景 |
| :------- | :--- | :------- |
| **Circular DMA** | 持续运行、尾部自动回绕、CPU 按写指针找新数据、须防覆盖 | 连续流（UART/音频/ADC） |
| **Normal DMA** | 一次处理一段、TC 后重新配置 | 分块传输 |
| **Ping-Pong / Double Buffer** | 两块交替、DMA 与 CPU 并行、无处理空窗 | 高吞吐显示/采样 |

---

## 🛑 7. 背压（Backpressure）与高低水位

当 `Producer > Consumer` 长期成立，真正有效的方向是：

> **让 Producer 暂停或降低发送速度——这就是背压（Backpressure）。**

```text
Producer ──► Buffer ──► Consumer
                │
     Buffer 达高水位 → 通知 Producer 暂停
     Buffer 降到低水位 → 允许 Producer 继续
```

### 7.1 高水位 / 低水位（滞回控制）

不能等 Buffer 100% 满了才发暂停信号，因为流控本身有延迟：

```text
Buffer 达高水位 ──► 发送 PAUSE ──► 信号传输 ──► 对端收到 ──► 对端停止发送
                        └──────── 这段时间数据仍在到达 ────────┘
```

因此必须预留**流控延迟期间的安全空间**，采用双阈值滞回：

```text
100% ──────────── Overflow（溢出，不可触及）
 80% ──────────── HIGH_WATERMARK → PAUSE
      安全余量 ↕
 40% ──────────── LOW_WATERMARK  → RESUME
  0% ──────────── Empty
```

> [!TIP] 阈值取值
> 高水位需满足：$(100\% - \text{HIGH}) \times C \ge R_p \times T_{\text{flowctl-delay}}$，即预留空间足以吸收"暂停信号生效前"仍在途的数据。典型取 80% PAUSE / 40% RESUME，滞回带避免频繁抖动。

### 7.2 ISR 在流控中的角色

ISR 不做大量协议解析，只做**水位检测 + 状态置位 + 通知**：

```text
ISR：Buffer ≥ HIGH_WATERMARK → flow_pause = true → 拉 RTS / 发 XOFF → 通知发送端暂停
Task：消费 Buffer → Buffer ≤ LOW_WATERMARK → flow_pause = false → 允许继续发送
```

### 7.3 对端是否支持流控，决定背压能否成立

| 对端能力 | 行为 | 结果 |
| :------- | :--- | :--- |
| **支持流控** | MCU 发 PAUSE → 上位机停止；Buffer 降 → RESUME → 上位机继续 | ✅ 完整背压闭环 |
| **不支持流控** | MCU 发 PAUSE，但上位机不认识、继续发送 | ❌ FIFO Overflow / Ring 被覆盖，仍丢数据 |

> [!CAUTION] 流控必须是双方共同遵守的机制
> 单方面宣布"我不接收了"**不能阻止对方继续发送**。若对端不支持流控，只能退而求其次：提高消费速度、增大 Buffer（延缓）、或按协议主动丢弃。

---

## 📡 8. 各外设的流控能力对比（UART / SPI / I²C）

判断 RX 是否容易溢出，**关键不是"有没有 DMA"，而是"接收端能不能控制 Producer 的产生速度"**。

### 8.1 UART RX：被动接收，需额外流控

```text
外部设备 ──► UART ──► DMA ──► RX Buffer ──► Parser
```

UART 没有"接收方控制数据时钟"的机制，一旦 `外部发送速度 > MCU 处理速度` 就会溢出。可选流控手段：

- **RTS/CTS** 硬件流控（最可靠）
- **XON/XOFF** 软件流控
- **ACK / Window** 协议层窗口
- 协议层限速、增大 Buffer、提高 Parser 消费速度

### 8.2 SPI Master RX：Master 控 SCK，天然背压

```text
MCU ──SCK──► Slave        Master 不产生 SCK，
MCU ◄─DATA── Slave        Slave 就无法输出下一位
```

Master 可"读 1 KB → 停 SCK → CPU 处理 → 再产生 SCK 读下一块"，**天然形成节流**。

### 8.3 SPI Slave RX：SCK 由外部控制，需额外流控

```text
外部 Master ──SCK/DATA──► MCU ──► DMA ──► Buffer
```

SCK 掌握在外部 Master 手里，若 Master 不顾 MCU 是否处理得过来持续供时钟，就会 `Buffer 满 → Overflow`。需要 **READY GPIO / 协议 ACK / Window-Credit / Master 查询状态** 等额外机制。

### 8.4 I²C Master RX：Master 控 SCL，天然节流

与 SPI Master 类似，Master 不产生 SCL，Slave 就无法输出后续数据，可"读一块 → 结束事务 → CPU 处理 → 再发起下一块"。

### 8.5 I²C Slave RX：Clock Stretching 硬件背压

I²C Slave 独有的 **Clock Stretching** 机制——Buffer 快满时把 SCL 拉低暂停时钟：

```text
Master                         Slave
  │──── SCL/SDA ────────────────►│
  │                          Buffer 快满
  │◄──── SCL 拉低（暂停时钟）─────│
  │                          CPU 处理数据、Buffer 下降
  │◄──── 释放 SCL ────────────────│
  │──── 继续 SCL ────────────────►│
```

> [!NOTE] Clock Stretching 的边界
> 它相当于**硬件级背压**，但只能暂停"当前正在进行的事务"，**不能禁止 Master 以后发起新事务**——下一笔事务仍由 Master 决定。

### 8.6 汇总对比

| 场景 | 谁控制数据产生节奏 | 接收端能否直接控制节奏 |
| :--- | :----------------- | :--------------------- |
| **UART RX**      | 外部设备 | ❌ 通常需要额外流控（RTS/CTS、XON/XOFF） |
| **SPI Master RX** | MCU Master | ✅ 可控制 SCK |
| **SPI Slave RX**  | 外部 Master | ❌ 需要额外流控（READY/ACK） |
| **I²C Master RX** | MCU Master | ✅ 可控制 SCL |
| **I²C Slave RX**  | Master（但 Slave 可 Clock Stretch） | ⚠️ 可暂停当前事务 |

---

## 🧠 9. 统一模型与决策清单

无论 UART、SPI、I²C，都先套用同一条数据链路：

```text
Producer ──► 通信外设 ──► DMA ──► Buffer ──► CPU / Parser ──► Consumer
```

据此按速率关系决策：

| 速率关系 | 系统状态 | 应对 |
| :------- | :------- | :--- |
| $R_p \le R_c$ | 正常 | 无需特殊处理 |
| $R_p > R_c$（短时突发） | Buffer 吸收突发 | 合理容量 Buffer 即可 |
| $R_p > R_c$（长期） | Buffer 最终一定会满 | **必须**从下方清单至少落实一条 |

长期不匹配时，必须至少做到以下之一：

```text
1. 降低 Producer 速度
2. 让 Producer 暂停（Backpressure）  ← 最根本
3. 提高 Consumer 处理速度
4. 增加缓存（只能延缓溢出，非根治）
5. 协议允许时，主动丢弃数据
```

> [!SUCCESS] 最关键的一条
> **DMA 解决"搬运效率"，Buffer 解决"短期速率不匹配"，背压解决"控制生产速度"。DMA、Ring Buffer、Double Buffer 本身都不能让一个长期"生产 > 消费"的系统凭空变得不溢出。**

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 设计 RX 链路前必过
> - [ ] **方向辨识**：明确 Producer / Consumer，判断接收端能否控制生产节奏
> - [ ] **TC 语义**：不用 DMA TC 当协议帧边界，改用 IDLE / CS / STOP 或长度+CRC
> - [ ] **速率核算**：估算 $R_p$、$R_c$，若长期 $R_p > R_c$ 必须上背压
> - [ ] **Buffer 模式**：连续流选 Circular + Ring；分块选 Normal；高吞吐选 Ping-Pong
> - [ ] **防覆盖**：Circular 模式做高水位检测，杜绝写指针追上读指针
> - [ ] **水位滞回**：设置 HIGH(PAUSE)/LOW(RESUME) 双阈值，预留流控延迟安全空间
> - [ ] **对端流控**：确认对端是否遵守 PAUSE/RESUME；不支持则提速或按协议丢弃
> - [ ] **ISR 轻量**：ISR 只检测水位、置状态、通知任务，重解析放下半部
> - [ ] **并发保护**：`dma_pos` / `read_pos` / `flow_pause` 主循环与 ISR 共访问，做临界区

---

## 🔗 思考与延伸索引

- [[SPI-DMA-多核-Cache]] — DMA 与 Cache 一致性、多核共享 Buffer 的同步问题
- [[modbus]] — RS-485 半双工下 DMA TC 与 UART TC 的"交接棒"时序，DE 切换踩坑
- [[UART-code]] — UART + DMA Circular + IDLE 中断的完整 RX 状态机实现
- [[Littlefs_FS]] — 存储侧同样存在的生产者/消费者与写入背压问题
- [[RTOS_Task_Priority]] — 提高 Consumer 消费速度：任务优先级与调度周期设计

---

## 📚 参考资料

1. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（DMA / USART / SPI / I2C 章节）.
2. **STMicroelectronics** — *AN4031: Using the STM32F4 DMA*（Circular / Double Buffer 模式）.
3. **ARM Developer** — *Cache Coherency and DMA* 应用笔记.
4. 项目内部笔记：[[modbus]]、[[SPI-DMA-多核-Cache]]。
