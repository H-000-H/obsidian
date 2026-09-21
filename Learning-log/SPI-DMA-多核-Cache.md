---
title: 高性能嵌入式全栈清单：SPI 物理层 · DMA 卸载 · 多核缓存一致性
tags:
  - Embedded
  - SPI
  - DMA
  - CacheCoherence
  - MultiCore
  - ARM
  - CortexM7
  - MemoryBarrier
status: Completed
created: 2026-09-20
---

# 📑 高性能嵌入式系统全栈清单：从物理层通信到多核缓存一致性

> [!INFO] 笔记概览
> - **主线**：一条 SPI 高速数据链路，从物理层信号 → 从机困境与 DMA 卸载 → Cache 一致性 → 伪共享防范 → 内存屏障 → 多核无锁交接，层层递进。
> - **内容范围**：SPI 四线拓扑与 CPOL/CPHA · 从机 Underrun/Overrun · DMA Burst · Write-back / Stale Read 冲突 · False Sharing · DMB/DSB/ISB · SPSC Ping-Pong 协议
> - **推荐路径**：§1 物理层 → §2 DMA 卸载 → §3 Cache 一致性 → §4 伪共享 → §5 屏障指令 → §6 多核交接 → §7 清单

---

## 🔌 1. SPI 物理层拓扑与硬件时序

### 1.1 四线物理接口与信号电气特性

| 信号 | 全称 | 方向 | 关键职责 |
| :--: | :--- | :--: | :------- |
| **SCLK** | Serial Clock | Master 独占驱动 | 决定传输波特率，典型可达数十 MHz |
| **MOSI** | Master Out Slave In | Master → Slave | 主机输出数据线 |
| **MISO** | Master In Slave Out | Slave → Master | 从机输出数据线；未选通（CS 高）时**必须高阻态（Hi-Z）**以支持多从机总线复用 |
| **CS/SS** | Chip Select / Slave Select | Master → Slave（低有效） | 从机寻址 + 帧同步（Frame Sync）+ 移位寄存器复位 |

### 1.2 环形移位寄存器（Ring Shift Register）模型

```text
   ┌──────────── Master ────────────┐
   │  ┌──────────────────────┐      │
MOSI├─►│  8/16-bit 移位寄存器 │◄─────┤ MISO
   │  └──────────────────────┘      │
   └───────────────┬────────────────┘
                   │ 首尾相连形成闭环
   ┌───────────────▼──── Slave ─────┐
   │  ┌──────────────────────┐      │
   │  │  8/16-bit 移位寄存器 │      │
   │  └──────────────────────┘      │
   └────────────────────────────────┘
```

> [!NOTE] "以发带收"机制
> SPI 硬件上**不存在纯接收或纯发送**。每个 SCLK 时钟沿，Master 移出 1 bit 同时移入 1 bit，Slave 亦然。因此主机若要读取从机数据，**必须主动发送 dummy byte**（如 `0xFF` / `0x00`）来提供时钟源。

### 1.3 四种工作模式（CPOL / CPHA 矩阵）

| 模式 | CPOL（空闲电平） | CPHA（采样边沿） | 数据输出时刻 (Drive) | 数据采样时刻 (Sample) |
| :--: | :-------------: | :-------------: | :------------------ | :------------------- |
| **Mode 0** | 0 (Low)  | 0 (1st Edge) | CS 下降沿 / SCLK 下降沿 | SCLK **上升沿** |
| **Mode 1** | 0 (Low)  | 1 (2nd Edge) | SCLK 上升沿 | SCLK **下降沿** |
| **Mode 2** | 1 (High) | 0 (1st Edge) | CS 下降沿 / SCLK 上升沿 | SCLK **下降沿** |
| **Mode 3** | 1 (High) | 1 (2nd Edge) | SCLK 下降沿 | SCLK **上升沿** |

> [!WARNING] 硬件设计隐患
> Mode 0 与 Mode 2（CPHA=0）在 **CS 拉低瞬间第 1 个 bit 必须立即呈现在数据线上**，对从机输出建立时间（Setup Time）要求极高。多从机或长走线场景优先选 CPHA=1（Mode 1/3）。

---

## ⚡ 2. 从机硬件困境与 DMA 异步卸载

### 2.1 从机的被动性与溢出风险

| 风险 | 触发条件 | 后果 |
| :--- | :------- | :--- |
| **Underrun（欠载）** | 从机 CPU 来不及往 Tx FIFO 填有效数据 | 移位寄存器对外发送垃圾数据 |
| **Data Overrun（覆盖）** | 从机 Rx FIFO 满，后续数据仍移入 | 数据被硬件丢弃/冲刷旧数据，置位 `SPI_SR_OVR` |
| **中断抖动 (Jitter)** | 高速（>20 MHz）下单字节传输 < 400 ns | 通用中断 Save/Restore 延迟达数百 ns~μs，**纯 CPU 中断模式必死** |

> [!CAUTION] 速率与中断延迟的硬约束
> 20 MHz SPI 传输 1 字节仅 $8 / 20\text{MHz} = 400\ \text{ns}$。而一次中断进出（压栈 + 向量跳转 + 出栈）通常 > 1 μs，纯中断逐字节搬运在高速场景根本无法跟上，必须上 DMA。

### 2.2 物理流控机制（Handshake / Ready Line）

引入专用硬件引脚（`BUSY` / `READY`）做带外握手：

```text
从机初始化完成 ──► 拉高 READY ──► Master 允许下发数据
从机处理/换包间隙 ──► 拉起 BUSY ──► Master（外部中断/轮询感知）停止驱动 SCLK
```

### 2.3 DMA 介入

- **外设与总线解耦**：SPI 的 Rx/Tx 请求线（Request Line）直接绑定 DMA 通道。
- **突发传输（Burst Transfer）**：配置增量地址传输，利用 FIFO 阈值触发 Burst，压低总线仲裁频率，把 CPU 从逐字节搬运中彻底解放。

---

## 🧱 3. 存储体系结构与缓存一致性（Cache Coherence）

现代 MCU（ARM Cortex-M7 / Cortex-A）引入哈佛结构的 I-Cache 与 D-Cache，DMA 与 CPU 访问同一物理 RAM 时产生一致性冲突。

### 3.1 冲突场景全景

```text
        [ Core 运算核心 ]
              ▲  │ 直接读写，命中 L1 D-Cache
              │  ▼
        [ L1 D-Cache ]  ← Snoop 仅在多核硬件总线支持时有效
              ▲  │ Clean / Invalidate（需显式软件指令维护）
              │  ▼
  ===== [ AXI / AHB 内部交叉总线矩阵 ] =====
         ▲                        ▲
         │ DMA 引擎直接读写物理 RAM  │ Core 穿透读取
         ▼                        ▼
   [ 物理 SRAM / TCM / 外挂 SDRAM ]
```

> [!CAUTION] 关键：DMA 绕过 Cache 直接访问物理 RAM
> CPU 走 Cache、DMA 走物理 RAM，两条路径对同一缓冲区的视图可能不一致，必须靠软件显式维护。

### 3.2 两类经典冲突与解法

| 冲突类型 | 场景 | 后果 | 解决方案 |
| :------- | :--- | :--- | :------- |
| **写回丢失**<br>(Write-back Hazard) | CPU 改缓冲区 A 仅停留在 D-Cache（Dirty）；DMA 发送物理 RAM 的 A | DMA 发出的全是旧数据 | 触发 DMA 前 **Cache Clean**：`SCB_CleanDCache_by_Addr` 把 Dirty 行刷回 RAM |
| **脏行覆盖**<br>(Stale Read / Dirty Overwrite) | DMA 把 SPI 数据灌入 RAM 缓冲区 B；CPU 读 B 命中 Cache 旧副本 | 读到过期数据，或 Cache 偶发写回踩死 DMA 新数据 | CPU 访问 B 前 **Cache Invalidate**：`SCB_InvalidateDCache_by_Addr` 清空 Valid 位 |

> [!TIP] 记忆口诀
> **发送前 Clean（把 CPU 的改动刷给 DMA 看），接收后 Invalidate（把 DMA 的新数据让 CPU 看见）。**

---

## 🛡️ 4. 伪共享（False Sharing）与空间安全规约

### 4.1 伪共享机制剖析

D-Cache 操作的最小单元是一行（Cache Line，通常 32 或 64 字节）。当 DMA 缓冲区与普通变量共处同一行时会误伤：

```text
        ┌──────── 64-byte Cache Line ────────┐
        │  X: DMA 缓冲区尾部  │  Y: 内核全局变量 │
        └────────────────────────────────────┘
CPU 刚更新 Y（Dirty，未写回） → DMA 收完 → CPU 对 X 执行 Invalidate
        → 整行 64 字节被抛弃 → Y 的最新写入值彻底蒸发 💥
```

> [!WARNING] 极难复现的内存错乱
> 这种"误伤失效"（Side-effect Invalidation）导致的 Bug 依赖变量布局与时序，表现为偶发、难复现的数据错乱，排查成本极高。

### 4.2 空间换安全的工程防范

**方案一：起始对齐 + 尺寸填充（Align & Pad）**

```c
#define CACHE_LINE_SIZE 64

/* 数据区独立占满若干整 Cache Line（256 是 64 的整数倍） */
typedef struct
{
    uint8_t payload[256];
} __attribute__((aligned(CACHE_LINE_SIZE))) spi_dma_buffer_t;

/* 状态控制结构体独占一行，填充至 64 字节 */
typedef struct
{
    volatile uint32_t flag;
    uint8_t reserved[CACHE_LINE_SIZE - sizeof(uint32_t)];  /* 填充 60 字节 */
} __attribute__((aligned(CACHE_LINE_SIZE))) cache_line_isolated_flag_t;
```

**方案二：MPU 硬件分区（最推崇）**

将专用通信环形区划入 MPU 指定 Region，内存属性配置为 `Device` 或 `Normal, Non-cacheable`：

> [!SUCCESS] 一劳永逸
> Non-cacheable 区域**彻底免疫 Cache 一致性问题**，省去所有 Clean/Invalidate 维护指令的开销。代价是该区域访问不走 Cache、单次延迟略高，适合 DMA 缓冲区这类"整块搬运、无随机复用"的场景。

---

## 🚦 5. ARM 体系结构屏障指令深度剖析

乱序发射、多级流水线、推测执行、写缓冲区（Store/Write Buffer）会让软件指令顺序与总线动作产生时间差。屏障指令用于强制同步。

### 5.1 屏障三兄弟全参数对比

| 指令 | 作用对象 | 硬件行为细节 | 经典底层场景 |
| :--: | :------- | :----------- | :----------- |
| **DMB** | 数据内存访问 | 保证之前的所有 Load/Store **在总线上被观测的顺序**先于其后的内存访问；流水线不暂停 | 多线程无锁队列、SMP 数据可见性保护 |
| **DSB** | 数据 + 流水线 | 挂起取指与执行，**等待 Store Buffer 清空、所有总线事务应答、所有 Cache 维护指令完成** | ① Cache Clean/Invalidate 之后 ② 启动外设 DMA 之前 ③ 触发 IPI（Mailbox/NVIC）之前 |
| **ISB** | 指令流水线 | **强行冲刷指令预取流水线**，丢弃已预取未执行指令，从 I-Cache/内存重载 | ① 切换特权级/栈指针（改 `CONTROL`）② 更新 MPU 属性或页表 ③ 运行刚写回的机器码 |

强度关系：$\text{DMB} < \text{DSB} < \text{ISB}$（从"仅约束内存序"到"清空总线事务"再到"刷新指令流水线"）。

> [!NOTE] 关键例外：异常返回无需手动 ISB
> Cortex-M 执行 `BX LR`（`LR` 为 `EXC_RETURN`）属于硬件定义的**上下文同步事件（Context Synchronization Event）**，硬件自动隐式刷新流水线，因此异常退出时无需手动追加 `ISB`。

---

## 🏓 6. 多核 Ping-Pong 无锁交互协议

单生产者-单消费者（SPSC）在异构/同构双核下的确定性时序设计。

### 6.1 内存实体定义与所有权模型

| 实体 | 说明 |
| :--- | :--- |
| **Buf_Ping / Buf_Pong** | 两块物理内存，置于共享 SRAM，**64 字节对齐** |
| **ping_ready / pong_ready** | 状态标志，配置为 **Non-cacheable** 区域，消灭 Cache 维护指令 |

所有权协议（以 `ping_ready` 为例）：

```text
ping_ready == 0  →  缓冲区归 Core 0（生产者）所有，Core 1 严禁读取
ping_ready == 1  →  缓冲区归 Core 1（消费者）所有，Core 0 严禁写入
```

### 6.2 完整无锁交接时序

```text
[ Core 0 (SPI DMA / 生产者) ]                [ Core 1 (算法核心 / 消费者) ]
             │                                          │
 1. 检查 ping_ready == 0（若为 1 则流控/等待）           │
 2. DMA 填充 Buf_Ping 完毕                              │
 3. SCB_CleanDCache(Buf_Ping)          📤              │
 4. ping_ready = 1（移交所有权）                        │
 5. __DSB()（排空 Store Buffer，确保标志落地）🛑         │
 6. 触发核间中断 IPI / Mailbox         🛎️ ────────────►│
 7. 立即切换填充 Buf_Pong              🔄       1. 进入 IPI 中断服务函数
             │                                 2. 检查 ping_ready == 1
             │                                 3. SCB_InvalidateDCache(Buf_Ping) 🧹
             │                                 4. __DSB()（确保失效完成，防乱序读）🛑
             │                                 5. 核心算法处理 Buf_Ping 数据
             │                                 6. ping_ready = 0（归还所有权）
             │                                 7. __DSB()（通知对端可见）🛑
             ▼                                          ▼
```

> [!TIP] 为什么每步标志写入后都要 `__DSB()`
> 若无 DSB，`ping_ready = 1` 可能仍滞留在 Store Buffer 中，而 IPI 已先到达对端，导致 Core 1 读到"标志未就绪但中断已来"的错乱状态。DSB 保证**标志落地先于中断可见**。

---

## ✅ 7. 落地校验清单（Checklist）

> [!SUCCESS] 高速 SPI + DMA + 多核链路上线前必过
> - [ ] **模式匹配**：Master/Slave 的 CPOL/CPHA 一致；高速多从机优先 CPHA=1
> - [ ] **MISO 高阻**：多从机总线确认未选通时 MISO 呈 Hi-Z
> - [ ] **DMA 卸载**：>数 MHz 场景禁用纯中断逐字节，改 DMA + Burst
> - [ ] **物理流控**：从机侧配置 READY/BUSY 握手，防 Underrun/Overrun
> - [ ] **发送前 Clean**：DMA TX 前 `SCB_CleanDCache_by_Addr`
> - [ ] **接收后 Invalidate**：DMA RX 后 CPU 访问前 `SCB_InvalidateDCache_by_Addr`
> - [ ] **对齐与填充**：DMA 缓冲区 64 字节对齐、尺寸补齐到 Cache Line 整数倍
> - [ ] **防伪共享**：DMA 缓冲区与普通全局变量不共处同一 Cache Line
> - [ ] **MPU 分区**（可选）：通信环形区设 Non-cacheable，免疫一致性问题
> - [ ] **屏障时序**：Cache 维护后、启动 DMA 前、发 IPI 前均插入 `__DSB()`
> - [ ] **标志可见性**：SPSC 所有权标志置于 Non-cacheable 区，写后 DSB 再通知对端

---

## 🔗 思考与延伸索引

- [[DMA、Buffer 与流量控制]] — DMA TC 语义、Circular/Ring Buffer、背压与高低水位（本清单的流控基础）
- [[SPI]] — SPI 协议基础：四线拓扑、CPOL/CPHA、全双工时序
- [[SPI-code]] — SPI + DMA 收发驱动参考实现
- [[嵌入式内核]] — 内存屏障、原子操作与无锁编程在内核中的应用
- [[C语言的宏魔法]] — `__attribute__((aligned))`、`volatile` 等编译器扩展与宏封装

---

## 📚 参考资料

1. **ARM** — *ARMv7-M Architecture Reference Manual*（DMB/DSB/ISB、Cache 维护指令章节）.
2. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（SPI / DMA / Cache / MPU）.
3. **STMicroelectronics** — *AN4838: Managing Cache Coherency on STM32H7*（Cache Clean/Invalidate 实践）.
4. **ARM Developer** — *False Sharing and Cache Line Alignment* 应用笔记.
