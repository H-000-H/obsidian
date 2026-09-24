# kernel 中断系统（ARM32）学习笔记

> 作用域：本篇覆盖 Linux 内核于中断入口与分发（`arch/arm/kernel/irq.c`）以及下半部（Bottom Half，SoftIRQ / Tasklet / Workqueue）两大块内容。它是《ARM Kernel head.S 内核启动 学习笔记》的中断相关延伸：内核在启动完成、`start_kernel()` 进入 C 语言虚拟世界后，中断子系统才逐步建立起来。

## 一、中断入口与分发（`arch/arm/kernel/irq.c`）

### 1. 控制中心 `struct irq_desc`

Linux 中**每个逻辑中断号的全局户口本**。它的三大职责：

- **向下**对接硬件控制器操作集（`irq_chip`：mask / unmask / ack / eoi 等底层回调）；
- **向上**连接外设驱动处理链表（`action`，多个 `irqaction` 串成链表共享同一个中断号）；
- **中间**掌管流控（`handle_irq`，常见有 `handle_level_irq`、`handle_edge_irq` 等，决定电平/边沿触发语义）。

### 2. `handle_IRQ` 的安全阀门

- 拦截硬件毛刺或越界的非法中断（避免死机），计入 `irq_err_count`；
- 合规中断查出 `desc`，移交给通用中断流控层继续处理。

### 3. 稀疏中断（`CONFIG_SPARSE_IRQ`）

- 摒弃低效的静态大数组，改用**基数树**按需动态分配 `irq_desc`；
- 通过 `arch_probe_nr_irqs` 动态设定中断号天花板；
- 大幅节省内存（不会为所有可能的最大中断号一次性铺满描述符数组）。

### 4. 核间中断（IPI，Inter-Processor Interrupt）

- 多核之间通过写 GIC 触发 SGI（Software Generated Interrupt）软中断，敲响目标核心的门铃；
- 配合共享内存队列完成任务：Reschedule（重新调度）、Call function（TLB 广播冲刷、函数调用）等跨核协作；
- 在 `arch_show_interrupts` 中统一向 `/proc/interrupts` 输出统计，便于排查。

---

## 二、下半部（Bottom Half）架构机制

Linux 将一次完整的中断处理切为两个半部：

- **上半部（Top Half / HardIRQ）**：关中断执行，仅做核心寄存器 ACK、应答硬件、登记事件后立即退出，要求快进快出。
- **下半部（Bottom Half）**：开中断执行，负责吞吐量大、耗时长的数据包解析、磁盘 I/O 排队、回调执行。

### 1. 下半部演进体系对比

Linux 提供了三种形态各异的下半部承载机制：

| 机制 | 运行上下文 | 并发特性 | 允许睡眠/阻塞？ | 适用场景 |
|---|---|---|---|---|
| **SoftIRQ（软中断）** | 中断上下文 | **极高并发**：同一类型的 SoftIRQ 可在多个 CPU 上同时并行执行（必须自备锁） | **严禁睡眠** | 吞吐与延迟要求极苛刻的核心子系统（网络、定时器、RCU） |
| **Tasklet** | 中断上下文 | **串行化保证**：基于 SoftIRQ 构建，同类型的 Tasklet 严禁跨核并发 | **严禁睡眠** | 普通驱动的快速下半部处理，免除跨核重入并发锁负担 |
| **Workqueue（工作队列）** | 进程上下文（内核线程 `kworker`） | 由普通进程调度器参与排队与轮转 | **允许睡眠/调度**（可拿 Mutex、做 I/O、分配内存） | 耗时长、需睡眠、需等待锁或大块内存分配的低速驱动逻辑 |

### 2. 标准 10 大软中断（SoftIRQ）位图与全景拆解

内核在 `include/linux/interrupt.h` 中严格定义了 10 种标准的软中断类型。每个 CPU 维护一个 32 位的 `__softirq_pending` 位图，每一个 bit 代表一个软中断类型（**编号越小，优先级越高**）：

```c
enum
{
	HI_SOFTIRQ=0,       /* 最高优先级 Tasklet */
	TIMER_SOFTIRQ,      /* 系统定时器驱动与回调 */
	NET_TX_SOFTIRQ,     /* 网络数据包发送队列清理 */
	NET_RX_SOFTIRQ,     /* 网络数据包协议栈接收与路由 */
	BLOCK_SOFTIRQ,      /* 通用块设备 I/O 完成处理 */
	IRQ_POLL_SOFTIRQ,   /* 块设备与高速存储轮询 (替代传统轮询) */
	TASKLET_SOFTIRQ,    /* 常规普通 Tasklet */
	SCHED_SOFTIRQ,      /* CFS 调度器周期性负载均衡 (SMP Load Balance) */
	HRTIMER_SOFTIRQ,    /* 高精度定时器 (已逐步合入硬时钟或按需启用) */
	RCU_SOFTIRQ,        /* 读-复制-更新 (RCU) 宽限期与回调批处理 */

	NR_SOFTIRQS
};
```

#### ① `HI_SOFTIRQ`（Bit 0）

- **本质**：高优先级 Tasklet。
- **机制**：优先于所有其他软中断处理，专供对延迟极度敏感的外设（如特定声卡驱动、低延迟总线控制器），确保其回调以最小的时钟抖动抢先完成。

#### ② `TIMER_SOFTIRQ`（Bit 1）

- **本质**：低精度系统定时器（Timer Wheel，时间轮）执行下半部。
- **机制**：由系统心跳滴答（Tick IRQ）在上半部触发 `raise_softirq(TIMER_SOFTIRQ)`。
- **职责**：
  - 推进动态定时器链表；
  - 执行驱动通过 `timer_setup()` / `mod_timer()` 注册的超时回调函数；
  - 驱动系统时间的软件更新与统计。

#### ③ `NET_TX_SOFTIRQ`（Bit 2）

- **本质**：网络发包清理（Transmit Cleanup）。
- **机制**：网卡发送队列通常采用环形缓冲区（Ring Buffer）。当网卡 DMA 把数据包推送到物理介质后，会发中断通知 CPU。
- **职责**：释放已成功发送的 `sk_buff` 结构体内存，解映射 DMA 缓冲区，唤醒被阻塞的上层发送队列，避免发送端 Socket 内存枯竭。

#### ④ `NET_RX_SOFTIRQ`（Bit 3）

- **本质**：网络收包核心总线（NAPI 轮询引擎）。
- **机制**：整颗 Linux 内核中**吞吐量最大、最容易触发软中断风暴**的类型。
- **职责**：
  - 驱动网卡 NAPI 机制的 `napi_poll()`；
  - 在开中断状态下批量从网卡 RX Ring Buffer 中抓取数据包，封装为 `struct sk_buff`；
  - 顺次推入链路层、IP 网络层（路由查找、Netfilter 防火墙）、TCP/UDP 传输层，直至压入用户进程 Socket 的接收队列。

#### ⑤ `BLOCK_SOFTIRQ`（Bit 4）

- **本质**：通用块设备（Block I/O）完成下半部。
- **机制**：当磁盘控制器（SATA/eMMC/NVMe）完成一次扇区 DMA 读写后，硬中断唤醒下半部。
- **职责**：执行 `blk_done_softirq()`，遍历完成的 `struct request`，通知上层文件系统（Ext4、Btrfs）或通用块层对应请求已就绪，触发 BIO 回调并唤醒等待该 I/O 的阻塞进程。

#### ⑥ `IRQ_POLL_SOFTIRQ`（Bit 5）

- **本质**：针对超高速存储介质的混合轮询架构。
- **机制**：应对 NVMe 等超高 IOPS 设备设计的接口（由 blk-iopoll 演进而来）。
- **职责**：在高 I/O 吞吐下，规避单次请求硬中断开销，转而借由下半部批量轮询存储控制器的完成队列，大幅压降超高并发存储下的中断负载。

#### ⑦ `TASKLET_SOFTIRQ`（Bit 6）

- **本质**：系统常规 Tasklet 处理池。
- **机制**：由 `tasklet_schedule()` 触发。
- **核心特性（免锁性保证）**：
  - 内核利用原子标志位 `TASKLET_STATE_RUN` 做出强行保证：**同一个 Tasklet 实例绝对不会在两个 CPU 核心上同时运行**；
  - 但**不同类型**的 Tasklet 可以跨核并发。
  - 这极大地简化了驱动编写门槛，驱动开发者无需为多核并发重入绞尽脑汁设计细粒度自旋锁。

#### ⑧ `SCHED_SOFTIRQ`（Bit 7）

- **本质**：多核 CFS 调度器负载均衡下半部（Load Balancing）。
- **机制**：在 SMP 多核系统中，各核的负载可能失衡（如 CPU 0 上排了 10 个进程，CPU 1 却空闲）。
- **职责**：
  - 在时钟中断后按需触发；
  - 遍历各调度域（Scheduling Domains）与调度组（Sched Groups）；
  - 计算各核心运行队列（Runqueue）的负载权重，必要时执行**跨核进程强制迁移**，平抑系统负载。

#### ⑨ `HRTIMER_SOFTIRQ`（Bit 8）

- **本质**：高精度定时器（High-Resolution Timers）软中断下半部。
- **机制**：早期为避免高精度定时器回调阻塞硬中断而引入。
- **演进**：现代高精度定时器机制已大幅优化，大多数硬实时高精定时任务直接在硬件时钟硬中断（HardIRQ）上下文执行完毕；仅在部分复杂的软超时或配置降级场景下，用该软中断做兜底批处理。

#### ⑩ `RCU_SOFTIRQ`（Bit 9）

- **本质**：读-复制-更新（Read-Copy Update）内存回收机制的核心推进器。
- **机制**：Linux 内核依赖无锁 RCU 实现高并发读取。写者更新数据后，旧数据不能立刻 `kfree()`，必须等待所有正在读取的 CPU 退出临界区（经历一个“宽限期” Grace Period）。
- **职责**：
  - 周期性检查本 CPU 是否已穿透静止状态（Quiescent State）；
  - 协同全局汇报宽限期结束；
  - 批量清空本核积压的 `call_rcu()` 回调链表，真正释放被淘汰的旧结构体内存。

### 3. 软中断执行引擎与防护屏障

软中断是一把双刃剑：它拥有直通硬件的高速性能，但如果不加节制，暴风雨般的网络包或外设事件会立刻让系统陷入“活锁（Livelock）”状态。

```
[外设硬中断上半部] ──> raise_softirq(nr) ──> 将当前 CPU 的 __softirq_pending 第 nr 位置 1
                                                        │
                                                        ▼
[退出硬中断: irq_exit()] ──> 检查 !in_interrupt() && local_softirq_pending()
                                                        │
                                                        ▼
[进入 do_softirq_own_stack()] ──> 汇编偷换 SP，切至独立中断栈 (irq_stack_ptr)
                                                        │
                                                        ▼
[核心引擎: __do_softirq()] ──> 循环处理位图 (最多 10 次 或 运行满 2ms)
         │
         ├── 未超标且已排空 ──> 换回原进程 SP，正常退出中断
         │
         └── 突发洪峰超标 (循环 >10次 或 >2ms)
                     │
                     ▼
             [wakeup_softirqd()]
                     │
                     ▼
          唤醒常驻单例 ksoftirqd/x
     (挂入 CFS 运行队列排队，降为普通分时任务)
```

#### ① 切栈防爆（`CONFIG_IRQSTACKS`）

- 为每个 CPU 分配专属独立中断栈（`irq_stack_ptr`）；
- 在 `do_softirq_own_stack` 中通过汇编偷换 SP；
- 防止下半部嵌套执行或深调用链压爆当前进程脆弱的 8KB 内核栈。

#### ② 抹 Flag 与优先就地执行（Local In-place Execution）

- **就地执行的极速优势**：硬中断刚结束，CPU 核心处于高度激活状态，驱动相关指令和数据正热烫地驻留在 L1/L2 Cache 中。就地借用当前上下文处理软中断，避开了极其昂贵的进程上下文切换、TLB 冲刷与 Cache 颠簸，实现了微秒级的极致网络吞吐和定时器精度。
- **原子清标志**：进入 `__do_softirq()` 后，一次性读出 32 位位图并将物理寄存器清零，开硬件中断逐 bit 遍历调用对应的 `action()` 业务函数，允许新的硬中断随时打入。

#### ③ 兜底降级：常驻线程 `ksoftirqd/x`

- **限额刹车**：同一个软中断上下文最多循环重试 **10 次**或执行满 **2 毫秒**。
- **平民化防假死**：一旦触发超时熔断，内核调用 `wakeup_softirqd()` 唤醒当前核绑定的 `ksoftirqd/x` 线程。
- **调度隔离**：`ksoftirqd` 在系统初始化时由 2 号进程（`kthreadd`）静态创建（每个核配且仅配一个，绝不动态创建）。它以普通的 `SCHED_NORMAL`（Nice = 0）身份挂入 CFS 红黑树，和终端 Shell、业务 App 平等均分时间片，并通过 `cond_resched()` 主动让渡 CPU。这保证了即使外设流量将网络打满丢包，操作系统的用户界面和调度系统也绝不假死。