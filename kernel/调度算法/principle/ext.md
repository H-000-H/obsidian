---
title: sched_ext 可扩展调度原理（Extensible Scheduler）
tags:
  - linux-kernel
  - scheduler
  - sched-ext
  - bpf
  - extensible-scheduling
  - dsq
status: Completed
created: 2026-10-07
---

# 🧬 sched_ext 可扩展调度原理（Extensible Scheduler）

> [!INFO] 笔记概览
> - **定位**：讲清"**把调度策略从内核搬到 BPF**"这一类可扩展调度要解决什么问题、用什么模型描述调度、以及为什么它能做到"**任意策略 + 系统不崩**"。**本篇只讲原理**，源码落地见 [[linux/ext|Linux sched_ext 实现与注意]]。
> - **内容范围**：为什么需要可扩展调度 · "内核提供机制、BPF 提供策略" · `sched_ext_ops` 事件模型 · DSQ（分派队列）· local/global/user 三类队列 · FIFO 与 vtime 排序 · 调度循环与 slice · bypass 兜底 · watchdog 与整体性 · 与经典调度类的对比 · 常见误区
> - **推荐路径**：§1 动机 → §2 核心思想 → §3 事件模型 → §4 DSQ → §5 调度循环 → §6 兜底机制 → §7 与经典类对比 → §8 误区

---

## 0. 一句话

sched_ext 是一个**调度类**，但它的行为**不写在内核里**，而是由一组 **BPF 程序**（"BPF 调度器"）定义。内核只保留**机制**（队列、时间片、迁移、兜底），把**策略**（谁先跑、跑多久、放哪个核）交给 BPF——从而在**不改内核、不重编译**的前提下试验任意调度算法，同时由内核保证**无论 BPF 调度器怎么写错，系统都不会死**。

---

## 1. 为什么要"可扩展调度类"

### 1.1 传统调度类的困境

Linux 的调度类（[[principle/stop|STOP]] / [[principle/deadline|DL]] / [[principle/rt|RT]] / [[principle/cfs|fair]] / [[principle/idle|IDLE]]）都是**内核里写死的策略**：

- 想试一个新算法（比如按缓存亲和度分组、按游戏/后台区分），**必须改内核源码、重编译、重启**；
- 就算改好了，**上游不一定收**，还得长期维护一个 out-of-tree 补丁；
- 更麻烦的是**无法针对工作负载动态切换**——调度器一启动就固定了。

### 1.2 已有的两条路，都不够

| 方案 | 做法 | 局限 |
|---|---|---|
| **改调度类源码** | 直接改 `fair.c` / `rt.c` | 门槛高、不可热切换、上游难接收 |
| **用户态调度** | `SCHED_OTHER` + 自己排线程优先级 | 无法控制内核线程、无法改选核/迁移、粒度粗 |
| **BPF 插桩** | 挂 tracepoint 观测 | **只能看，不能改**调度决策 |

于是就有了第三个选择：**让内核把调度决策本身也做成可编程的**。

---

## 2. 核心思想：机制与策略分离

sched_ext 的设计目标（见 `Documentation/scheduler/sched-ext.rst`）：

1. **导出完整的调度接口**——任何调度算法都能在它之上实现；
2. **CPU 分组自由**——任务在唤醒时**不绑定具体 CPU**，BPF 调度器可随意把 CPU 编组一起调度；
3. **可动态开关**——BPF 调度器能随时加载/卸载，运行中切换；
4. **系统整体性优先**——无论 BPF 调度器做什么，一旦出错、任务卡住、或按 `SysRq-S`，**立刻回退到默认（fair）调度**。

一句话概括这个分工：

> **内核提供"机制"**：队列结构、时间片记账、选核入口、迁移原语、看门狗、回退路径。
> **BPF 提供"策略"**：`enqueue` / `dispatch` / `select_cpu` 等回调里的判断逻辑。

这与"微内核把策略放用户态"的思路同源，但**策略跑在内核内的 BPF 沙箱里**，因此**没有用户态调度的上下文切换与信息缺失问题**。

---

## 3. 事件模型：`sched_ext_ops`

BPF 调度器通过一组回调（`struct sched_ext_ops`）**被动接收内核通知**，并在回调里做决策。理解 sched_ext 的关键，是理解**这些回调什么时候被调用**。

### 3.1 任务生命周期回调（成对出现）

| 回调 | 触发时机 | 作用 |
|---|---|---|
| `select_cpu(p, prev_cpu, wake_flags)` | 任务**唤醒**时 | 选一个 CPU（可返回 `-1` 表示"随便"） |
| `enqueue(p, enq_flags)` | 任务**进入就绪** | 决定把任务放进哪个 DSQ（或直接放本地） |
| `dequeue(p, deq_flags)` | 任务**离开就绪**（睡眠/迁移） | 通知从 BPF 手里收回 |
| `dispatch(cpu, prev)` | CPU **要找下一个任务**时 | 从自有 DSQ 搬任务到本地 DSQ |
| `runnable(p)` / `running(p)` / `stopping(p)` / `quiescent(p)` | 就绪↔运行 的**四态转换** | 精细化观测任务状态变化 |
| `yield(from, to)` | `sched_yield()` | 允许拒绝/改写让出 |
| `set_weight` / `set_cpumask` / `update_idle` | 属性/idle 变化 | 同步内部记账 |
| `init_task` / `exit_task` | 任务进入/离开 sched_ext | 初始化/回收 BPF 侧状态 |

> 关键点：**内核只在这些"事件点"回调 BPF**，BPF **不能**主动去"抢占式地重新规划整个世界"。这种"事件驱动"模型让内核始终掌握主动权，是系统安全的根基。

### 3.2 四种状态转换（`runnable`/`running`/`stopping`/`quiescent`）

```
        ┌──────────── runnable ────────────┐
        │                                   │
   quiescent                          ┌──► running ──┐
        ▲                              │             │
        │                          running        stopping
        └──────────── stopping ◄──────┘             │
              (runnable=false)                      │
```

- **runnable**：变就绪（可能还没上 CPU）；
- **running**：真正开始跑；
- **stopping**：被换下但**仍就绪**（马上还会被调度）；
- **quiescent**：不再就绪（睡眠/退出）。

**runnable↔quiescent 包含 running↔stopping**——这个"嵌套"关系保证 BPF 看到的"就绪区间"是完整、连续的。

### 3.3 ops flags（行为开关）

调度器加载时用一组 `SCX_OPS_*` 标志声明自己的假设，例如：

| 标志 | 含义 |
|---|---|
| `SCX_OPS_SWITCH_PARTIAL` | **只**接管 `SCHED_EXT` 任务，其余仍归 fair |
| `SCX_OPS_ENQ_LAST` | 切片用尽且无其他任务时，把当前任务也交回 `enqueue()` |
| `SCX_OPS_ENQ_EXITING` | 退出中的任务也走 `enqueue()`（默认自动丢本地队列） |
| `SCX_OPS_ENQ_MIGRATION_DISABLED` | 不可迁移任务也走 `enqueue()` |
| `SCX_OPS_ALLOW_QUEUED_WAKEUP` | 允许"排队唤醒"优化（跳过部分 `select_cpu`） |
| `SCX_OPS_KEEP_BUILTIN_IDLE` | 即使实现了 `update_idle` 也保留内置 idle 跟踪 |

> 未设 `SWITCH_PARTIAL` 时，加载后 **`SCHED_NORMAL`/`SCHED_BATCH`/`SCHED_IDLE`/`SCHED_EXT` 全部**交给 sched_ext；设了则只接管 `SCHED_EXT`。

---

## 4. DSQ：分派队列

DSQ（**dispatch queue**）是 sched_ext 的**核心数据结构**，也是它相对传统调度类最大的不同。

### 4.1 为什么需要 DSQ

传统调度类里，任务在**每 CPU 的就绪队列**上排队。sched_ext 里，任务**唤醒时不绑定 CPU**——它先进 BPF 调度器自己的队列，等 CPU 需要时再被"分派"过去。于是需要一种**在核心与 BPF 之间缓冲任务的队列**，这就是 DSQ。

### 4.2 三类 DSQ

| 类型 | 谁拥有 | 说明 |
|---|---|---|
| **Local DSQ**（每 CPU 一个） | 内核核心 | CPU 真正**从这里取任务运行**；BPF 最终必须把任务放进来 |
| **Global DSQ**（`SCX_DSQ_GLOBAL`） | 内核核心 | 全局兜底队列；按 NUMA 节点分片以扩展性 |
| **User DSQ**（BPF 自建） | BPF 调度器 | 任意数量，BPF 用 `scx_bpf_create_dsq()` 创建、自定义语义 |

**数据流**：

```
 唤醒 ──► ops.enqueue() ──► [User DSQ / Global DSQ]  ← BPF 自由编排
                                   │
                          ops.dispatch(cpu) 或
                          scx_bpf_dsq_insert(SCX_DSQ_LOCAL)
                                   │
                                   ▼
                          Local DSQ ──► CPU 取任务运行
```

### 4.3 两种排序：FIFO 与 vtime

| 排序 | 依据 | 用途 |
|---|---|---|
| **FIFO** | 入队顺序（链表） | 简单队列；内置 DSQ **只**支持 FIFO |
| **vtime 优先队列（PRIQ）** | `p->scx.dsq_vtime`（红黑树） | 实现"虚拟时间公平"类算法（类似 CFS/EEVDF 的思路） |

- 内置的 Local / Global DSQ **强制 FIFO**——避免"FIFO 任务饿死 vtime 任务"；
- PRIQ 插入时，任务**同时**挂红黑树和链表，保持 `list` 也是 vtime 有序。

### 4.4 任务归属状态机（`ops_state`）

因为任务在"核心 ↔ BPF"之间来回交接，需要一个**归属状态机**来判定"此刻谁拥有这个任务"：

```
      .────────────► NONE（核心拥有）
      │               │         ▲
      │      enqueue  │         │ 直接分派
      │               ▼         │
      │          QUEUEING ──────┘
      │               │
      │      enqueue  │
      │      完成     ▼
      │           QUEUED（BPF 拥有）
      │               │
      │     dispatch  │
      │               ▼
      └──────────  DISPATCHING（交还核心中）
```

- `NONE`：核心拥有（可入队/被抢占）；
- `QUEUEING`：正在交给 BPF（短暂瞬态）；
- `QUEUED`：**BPF 拥有**——此时任务在 BPF 的 DSQ 上，BPF 可随时 `dispatch` 它；
- `DISPATCHING`：正在从 BPF 交还核心。

每个 `QUEUED` 还带一个 **qseq（序号）**：当 dispatch 与 dequeue/重新入队**并发竞争**时，靠 qseq 判断"我这次 dispatch 是否还对得上那个任务"，从而**安全地忽略过期分派**。这套机制让 BPF 调度器**可以随意、甚至"错误地"分派任意任务**，内核负责拒绝无效操作——**大幅简化 BPF 侧实现**。

---

## 5. 调度循环与时间片

### 5.1 一个 CPU 找下一个任务（pick）

```
CPU 需要选下一个任务
        │
        ▼
 本地 DSQ 有任务？──是──► 取队首运行
        │否
        ▼
 ops.dispatch(cpu, prev)   ← 让 BPF 从自有 DSQ 搬任务过来
        │
        ▼
 搬到本地 DSQ 了吗？──是──► 取队首运行
        │否
        ▼
 保持 prev 继续跑（切片够的话）／否则让 CPU 空闲
```

要点：

- **CPU 只从 Local DSQ 取任务运行**——这是内核与 BPF 的硬边界；
- `ops.dispatch()` 是 BPF **唯一的"主动补货"入口**，但它**每次只对当前 CPU** 生效，不改变别处；
- 若 BPF 没搬来任务且 `prev` 还有切片，就**继续跑 `prev`**（减少切换开销）。

### 5.2 时间片（slice）

- 每个任务有一个 `p->scx.slice`（纳秒）；
- **BPF 在入队时分派**：`scx_bpf_dsq_insert(p, dsq, slice, flags)` 里给切片；
- 任务运行时**切片递减**，**耗尽即触发调度事件**（重新选队）；
- 若 BPF 忘了给切片，内核用**默认切片**（`SCX_SLICE_DFL` = 20ms）兜底；
- `SCX_SLICE_INF` 表示无限切片（隐含 nohz，不再需要 tick）。

> 切片是 sched_ext 里 BPF 控制"运行时长"的**唯一手段**；它**不等于**优先级，只决定"让出前能跑多久"。

### 5.3 抢占

- 内核在合适事件点调用 `wakeup_preempt`，判断新任务是否应抢占当前任务；
- BPF 可通过 **`SCX_ENQ_PREEMPT`** 主动请求"插入并抢占"（内核把当前任务切片清零、触发 resched）；
- BPF 还可用 `scx_bpf_kick_cpu()` **踢醒**某个 CPU 去重新调度（例如它刚往那个 CPU 的本地 DSQ 放了任务）。

---

## 6. 兜底机制：为什么"系统不会崩"

这是 sched_ext 能安全上生产的关键。设计原则：**永远假设 BPF 调度器会犯错**。

### 6.1 bypass（旁路）模式

当 sched_ext 正在**启用/停用**、或检测到问题时，进入 **bypass**：

- 所有任务被放进 **bypass DSQ**，用**固定短切片**（`SCX_SLICE_BYPASS` = 5ms）**轮转**；
- 此时**不再回调 BPF**（不信任它），只求"所有任务都能前进"；
- bypass 会**沿调度器树向下传播**（子调度器也跟着 bypass）。

### 6.2 watchdog（看门狗）

- 每个"就绪但长时间没被调度"的任务会被记录；
- 超过 `watchdog_timeout`（通常秒级）仍未运行 → **判定 stall** → **踢掉 BPF 调度器**并回退 fair；
- 另有一个"**看门狗自己**没按时检查"的二级看门狗（防止看门狗线程本身卡死）。

### 6.3 其他回退触发

| 触发 | 结果 |
|---|---|
| BPF 调 `scx_bpf_error()` | 立即 abort + 回退 fair，并 dump 调试信息 |
| 任务**反复被重新入队却始终不运行**（reenq 超限） | abort（BPF 在"反复决定一个它做不到的安置"） |
| RCU CPU stall / soft lockup | 关联到 sched_ext 时 abort |
| `SysRq-S` | 手动 abort 并回退 |
| CPU 热插拔但 BPF 未实现 `cpu_online/offline` | 退出并带 `RESTART` 意图（重启调度器） |

### 6.4 回退路径本身也要可靠

abort 后，所有任务必须**安全地回到 fair 类**。因此内核维护一份**独立于 PID 的"全部 sched_ext 任务"清单**（任务可能在退出中已丢失 PID），确保**任何状态下都能遍历并搬回**。

---

## 7. 与经典调度类的对比

| 维度 | 经典调度类（RT/CFS/DL） | **sched_ext** |
|---|---|---|
| 策略位置 | 内核源码（编译期固定） | **BPF 程序（可动态加载/卸载）** |
| 能否热切换 | 不能 | **能**（运行中 enable/disable） |
| 任务排队 | 每 CPU 就绪队列 | **DSQ**（唤醒不绑 CPU） |
| 优先级模型 | 固定（数字/截止期/vruntime） | **完全自定义**（BPF 决定） |
| 时间片 | 内核规则（如 RR 100ms） | **BPF 指定**（缺省 20ms） |
| 选核 | 内核 `select_task_rq` | **BPF `select_cpu`** |
| 安全兜底 | 无此概念 | **bypass + watchdog + 回退 fair** |
| 调试 | 改代码重编译 | dump + `events` 计数器 + tracepoint |
| 代价 | 无额外开销 | **BPF 回调开销 + 每个决策多一层** |

**本质区别**：经典调度类回答"**这类任务该怎么排**"（模型固定）；sched_ext 回答"**如何让任何人自己定义怎么排**"（机制开放）。前者胜在**确定性、零开销**，后者胜在**可实验性、可定制、可热插拔**。

---

## 8. 常见误区

1. **"sched_ext 是一个具体调度算法"**？不是。它是一个**框架/机制**；具体算法由每个 BPF 调度器实现（如 `scx_simple`、`scx_lavd` 等）。
2. **"加载后只影响 `SCHED_EXT` 任务"**？默认**不是**——除非设了 `SCX_OPS_SWITCH_PARTIAL`，否则普通任务**全部**被接管。
3. **"BPF 可以随便改任何任务"**？不能。BPF **只能通过事件回调**介入，且分派受 `ops_state`/qseq 校验，**无效分派被内核忽略**。
4. **"BPF 分派到哪个 DSQ 就立刻上 CPU"**？不是。**CPU 只从 Local DSQ 取任务**；进 User/Global DSQ 只是"排队"，还要等 `dispatch()` 搬过来。
5. **"切片就是优先级"**？不是。切片只控制"让出前能跑多久"；**谁先跑**由 DSQ 顺序/BPF 逻辑决定。
6. **"sched_ext 崩了系统就崩"**？恰恰相反——**bypass + watchdog + 回退 fair** 就是为"BPF 写错也不死机"而设计的。
7. **"bypass 时还会回调 BPF"**？不会。bypass 的语义正是"**不信任 BPF**，用固定规则让所有任务前进"。
8. **"可以完全替代 CFS"**？可以接管全部任务，但**性能与正确性由 BPF 调度器负责**；框架本身不保证调度质量。

---

## 9. 一句话总结

> **sched_ext = 内核保留机制（DSQ、切片、迁移、bypass、watchdog），把策略（谁先跑、跑多久、放哪核）交给 BPF 程序，通过 `sched_ext_ops` 事件回调 + `ops_state` 归属状态机 + Local DSQ 硬边界，实现"任意算法可热插拔、且 BPF 写错也不会拖垮系统"的可扩展调度类。**

---

## 🔗 相关

- [[linux/ext|Linux sched_ext 实现与注意]]：`kernel/sched/ext/` 的源码级细节（DSQ、dispatch、bypass、watchdog、子调度器）
- [[principle/cfs|CFS 完全公平调度原理]]：公平调度的经典模型（vtime 思路与 DSQ 的 PRIQ 相通）
- [[principle/rt|RT 实时调度原理]]：固定优先级实时调度，与"可编程策略"形成对照
- [[principle/stop|STOP 调度类原理]]、[[principle/idle|IDLE 调度类原理]]、[[principle/deadline|截止期调度原理]]