---
title: CFS 完全公平调度原理（含 EEVDF 演进）
tags:
  - linux-kernel
  - scheduler
  - cfs
  - eevdf
  - vruntime
  - fair-scheduling
status: Completed
created: 2026-10-06
---

# ⚖️ CFS 完全公平调度原理（含 EEVDF 演进）

> [!INFO] 笔记概览
> - **定位**：讲清"完全公平调度"这一类算法要解决什么问题、用什么模型刻画公平，以及 Linux 从 **CFS**（单指标 vruntime）演进到 **EEVDF**（vruntime + 虚拟截止期双指标）的动因。**本篇只讲原理**，Linux 落地细节与源码见 [[linux/cfs|Linux CFS/EEVDF 实现与注意]]。
> - **内容范围**：公平的度量与权重模型 · 虚拟运行时间 vruntime · 红黑树最左选择 · 经典 CFS 的时间片与抢占 · 经典 CFS 的公平性缺陷 · EEVDF 的 lag / 合格性 / 虚拟截止期 · 与 EDF、DL 的本质区别 · 组调度与带宽限流
> - **推荐路径**：§1 公平与权重 → §2 vruntime → §3 经典 CFS → §4 缺陷 → §5 EEVDF → §6 对比与边界

---

## 🎯 1. 什么叫"完全公平"

### 1.1 公平的度量：按权重分 CPU

普通分时调度要回答一个问题：**CPU 时间怎么分？** CFS 的答案是"**按权重（weight）比例分**"。

- 每个任务有一个 `nice` 值（-20 ~ +19），映射到一张**权重表**；
- `nice = 0` 的权重记为 `NICE_0_LOAD`（内核里是 1024）；
- nice 越小（优先级越高）权重越大，理论 CPU 份额越大。

于是"公平"被定义成一个可计算的量：

$$
\text{任务 } i \text{ 的份额} = \frac{w_i}{\sum_j w_j}
$$

> [!NOTE] 公平 ≠ 优先级抢占
> CFS 里没有"谁优先级高谁独占 CPU"这回事。高 nice（大权重）任务只是**分到更多份额**，而不是"来了就把别人踢走"。这与 RT / DL 的抢占语义完全不同。

### 1.2 理想模型：完美多任务

CFS 的设计灵感来自一个理想化模型——**如果 CPU 快到可以同时运行所有任务**，那么每个任务都以 $1/n$ 的速度推进，人人都不被延迟。

真实 CPU 一次只能跑一个，于是 CFS 用"**虚拟时间**"来逼近这个理想：让每个任务的"虚拟进度"增长得一样快，谁落后谁先跑。

---

## 🧮 2. 虚拟运行时间 vruntime

### 2.1 定义

CFS 为每个可调度实体维护一个 **vruntime（虚拟运行时间）**：

$$
\Delta \text{vruntime}_i = \Delta t \cdot \frac{\text{NICE\_0\_LOAD}}{w_i}
$$

- 权重 $w_i$ 越大 → 同样的真实时间 $\Delta t$，vruntime 增长**越慢**；
- 权重 $w_i$ 越小 → vruntime 增长**越快**。

### 2.2 为什么这样就"公平"

设两个任务都持续可运行，调度器总让 **vruntime 最小**的任务跑。稳态下两者的 vruntime 会交替领先，即近似：

$$
\frac{C_1}{w_1} = \frac{C_2}{w_2}
\;\Longrightarrow\;
\frac{C_1}{C_2} = \frac{w_1}{w_2}
$$

也就是说，**获得的 CPU 时间严格正比于权重**。这就是"完全公平"四个字的由来。

### 2.3 数据结构：红黑树 + 最左节点

- 所有可运行实体挂在**以 vruntime 为键的红黑树**上；
- 选下一个任务 = 取**最左节点**（vruntime 最小），$O(\log n)$；
- 新任务入队、当前任务更新 vruntime 后重新插入，维持有序。

> [!TIP] 用"虚拟时间"代替"真实时间"做排序键
> 这是 CFS 最核心的一招：把"按权重分配"这件复杂的事，化归为"永远选虚拟时间最小的那个"。排序结构（红黑树）不需要知道权重，权重只在计算 $\Delta$vruntime 时体现。

---

## ⏱️ 3. 经典 CFS（2.6.23 ~ 6.5）：时间片与抢占

### 3.1 调度周期与时间片

经典 CFS 用两个全局可调参数界定节奏：

| 参数 | 默认值 | 含义 |
|---|---|---|
| `sched_latency` | 6 ms | 一个"调度周期"，理想上让每个可运行任务都跑一次 |
| `sched_min_granularity` | 0.75 ms | 单次运行的最小粒度，防止切换过于频繁 |

当可运行任务数 $n$ 超过 $\text{sched\_latency} / \text{min\_granularity}$（默认 8）时，周期被拉长为：

$$
\text{period} = n \times \text{sched\_min\_granularity}
$$

每个任务的时间片按权重分配：

$$
\text{sched\_slice}_i = \text{period} \times \frac{w_i}{\sum_j w_j}
$$

### 3.2 抢占时机

- **tick 抢占**：当前任务的 vruntime 落后于树中最左实体超过 `sched_wakeup_granularity` 时，标记 `TIF_NEED_RESCHED`；
- **唤醒抢占**：新唤醒任务的 vruntime 比当前任务小得足够多时抢占；
- **min_vruntime**：`cfs_rq` 维护一个单调不减的 `min_vruntime`，新任务/唤醒任务的 vruntime 被"垫高"到它附近，防止长期睡眠的任务一醒来就霸占 CPU（**sleeper fairness** 启发式）。

> [!WARNING] sleeper fairness 是"补丁"而非"原理"
> 把一个睡眠很久的任务的 vruntime 抬到 `min_vruntime`，本质是"让它不要占便宜"。这是一个启发式补丁，说明**纯 vruntime 模型无法正确表达"睡眠期间应得的补偿"**。EEVDF 用 lag 把这件事变成了一等公民。

---

## ⚠️ 4. 经典 CFS 的两个结构性缺陷

### 4.1 vruntime 公平 ≠ 时延保证

CFS 只保证**长期比例公平**，它对"任务这一次想跑多久"一无所知。考虑两个权重相同的任务：

- **A**：批处理，希望一次连续跑 10 ms；
- **B**：交互，希望每次只跑 0.5 ms 但要求**尽快**被调度。

CFS 给两者相同的时间片，结果：**A 被切得七零八落**（吞吐下降），**B 也得不到"请求越短、响应越快"的时延优势**（延迟没有变好）。这就是"公平"与"时延"两回事。

### 4.2 缺少"请求大小"这个维度

CFS 的排序键只有一个（vruntime），所以它**无法区分请求大小**。而"短请求应获得更低的调度延迟"是分时系统的基本诉求。

→ 这两点直接催生了 **EEVDF**。

---

## 🚀 5. EEVDF：合格性 + 虚拟截止期

> **Earliest Eligible Virtual Deadline First**（最早合格虚拟截止期优先），Linux 6.6 起取代经典 CFS 的核心算法。名字里的 "Earliest ... First" 与 EDF 同源，但**截止期是虚拟的**。

### 5.1 两个判据

EEVDF 对每个实体 $(v_i, w_i)$ 引入两个概念：

**(1) lag（滞后量）——衡量"欠了多少服务"**

$$
\text{lag}_i = S_i - s_i = w_i\,(V - v_i)
$$

其中 $V$ 是全体实体的**加权平均虚拟时间**：

$$
V = \frac{\sum_j v_j w_j}{\sum_j w_j}
$$

- $\text{lag}_i > 0$：这个实体**被亏欠**了服务；
- $\text{lag}_i < 0$：这个实体**占了便宜**（跑多了）。

**(2) 合格（eligible）**

$$
\text{eligible}_i \iff \text{lag}_i \ge 0 \iff V \ge v_i
$$

只有"还欠着服务"的实体才有资格被选中。

### 5.2 虚拟截止期

每个实体还有一个**请求大小** $r_i$（即时间片 slice）。虚拟截止期定义为：

$$
vd_i = \max(v_i,\ V) + \frac{r_i}{w_i}
$$

直觉：**从"当前该到的时间点"起，往后加一段与请求大小成正比、与权重成反比的虚拟时间**。请求越大，截止期越远；权重越大，截止期越近。

### 5.3 调度规则

> 在所有**合格**的实体中，选**虚拟截止期 $vd_i$ 最小**的那个。

- **公平性**由"合格性"保证：只选被亏欠的，不选占便宜的；
- **时延**由"虚拟截止期"体现：请求小的实体 $vd$ 更近，更快被选中 → **小请求低延迟**；
- 红黑树以 $vd$ 排序，同时用**增广字段**（子树最小 $v$）做剪枝，仍是 $O(\log n)$。

### 5.4 lag 的稳态界

EEVDF 给出稳态下 lag 的紧界：

$$
-r_{\max} < \text{lag}_i < \max(r_{\max},\ q)
$$

（$r_{\max}$ 为最大请求/时间片，$q$ 为与 tick 相关的粒度）。这说明**任何任务的"欠/欠账"都被限制在一个时间片量级内**——既不会无限被亏欠（公平），也不会无限占便宜。

> [!NOTE] 基准时间片变小了
> 经典 CFS 的 `sched_latency` 是 6 ms；EEVDF 改用单一的 `sched_base_slice`，默认 **0.7 ms**。请求粒度更细，配合虚拟截止期能更好地表达时延需求。

---

## 🔍 6. 对比与边界

### 6.1 EEVDF vs EDF vs DL（别混淆）

| | EEVDF（fair 类） | EDF / DL（deadline 类） |
|---|---|---|
| 截止期性质 | **虚拟**，由 vruntime 派生 | **真实**，来自任务参数 $(Q,D,P)$ |
| 保证 | 长期比例公平 + 请求相关时延 | **可证明**的截止期满足 |
| 参数来源 | nice / 权重 | runtime / deadline / period |
| 准入控制 | 无 | **有**（带宽准入） |
| 超载行为 | 按权重退化，不崩溃 | 需 CBS 隔离，否则 domino effect |

> [!WARNING] 名字像，本质不同
> EEVDF 的 "Virtual Deadline" 是**公平性工具**，不是实时承诺。真正的截止期调度见 [[principle/deadline|截止期调度原理]]。

### 6.2 公平 ≠ 限流：CFS 带宽控制

`CONFIG_CFS_BANDWIDTH`（`cpu.max`）给一组任务设"每周期最多用多少 CPU 时间"，超出即**throttle**。

- **CFS 公平**：在**可运行任务之间**按权重分时间——解决"怎么分"；
- **CFS 带宽**：限制**一组任务整体**的时间上限——解决"总共给多少"。

两者正交：一个被限流的组内部依然按权重公平分配。

### 6.3 组调度（task_group）

当启用 `CONFIG_FAIR_GROUP_SCHED`，**任务组本身也作为一个实体**参与竞争：

- 先在组之间按权重公平；
- 再在组内部按权重公平；
- 递归下去 → 无论进程怎么分组，CPU 份额都按层级权重分配。

> 这是容器/cgroup CPU 份额的底层机制：`cpu.weight` 控制的就是这里。

### 6.4 一句话总结

> **CFS = 用 vruntime 把"按权重分 CPU"化归为"总选最左"；EEVDF 在此基础上补上"请求大小"维度——用 lag 判合格、用虚拟截止期排先后，既保住公平，又给了短请求低延迟。**

---

## 🔗 相关

- [[linux/cfs|Linux CFS/EEVDF 实现与注意]]：`fair.c` 的源码级细节
- [[principle/deadline|截止期调度原理]]：真实截止期与 CBS/GRUB
- [[principle/stop|STOP 调度类原理]]、[[principle/idle|IDLE 调度类原理]]
