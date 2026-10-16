---
title: Linux DL（SCHED_DEADLINE）实现与注意
tags:
  - linux-kernel
  - scheduler
  - sched-deadline
  - edf
  - cbs
  - grub
  - bandwidth
status: Completed
---

# 🎯 Linux DL（`SCHED_DEADLINE`）实现与注意

> [!INFO] 笔记概览
> - **源码基线**：`torvalds/linux` master，**Linux 7.3.0-rc6**，commit `22430ae5d90a`（2026-10-06）。文件：`kernel/sched/deadline.c`、`kernel/sched/sched.h`、`include/linux/sched.h`。
> - **定位**：把 [[principle/deadline|截止期调度原理]] 落到 Linux 真实实现——EDF 选任务 + CBS 限带宽 + GRUB 回收空闲带宽。本篇给出**关键数据结构、核心函数与源码位置**，并列出容易踩的坑。
> - **内容范围**：调度类层级与文件地图 · `sched_dl_entity` / `dl_rq` / `dl_bw` 数据结构 · 入队出队 · 预算记账与 GRUB 公式 · 节流与补充 · 0-lag 延迟回收 · 准入控制 · **DL server**（fair/ext 的带宽服务器）· 参数校验 · 注意点
> - **推荐路径**：§1 层级与地图 → §2 数据结构 → §3 预算记账 → §4 节流/补充 → §5 准入 → §6 DL server → §7 注意点

---

## 🗺️ 1. 调度类层级与文件地图

### 1.1 优先级顺序（源码确定）

顺序由链接脚本 `include/asm-generic/vmlinux.lds.h` 的 `SCHED_DATA` 段决定，并由 `core.c` 的 `BUG_ON` 断言：

```
stop > dl > rt > fair > ext > idle
```

```c
/* include/asm-generic/vmlinux.lds.h */
__sched_class_highest = .;
*(__stop_sched_class)
*(__dl_sched_class)
*(__rt_sched_class)
*(__fair_sched_class)
*(__ext_sched_class)
*(__idle_sched_class)
__sched_class_lowest = .;
```

`core.c` 中的自检：

```c
BUG_ON(!sched_class_above(&stop_sched_class, &dl_sched_class));
BUG_ON(!sched_class_above(&rt_sched_class, &fair_sched_class));
BUG_ON(!sched_class_above(&fair_sched_class, &idle_sched_class));
BUG_ON(!sched_class_above(&fair_sched_class, &ext_sched_class));
BUG_ON(!sched_class_above(&ext_sched_class, &idle_sched_class));
```

> DL 排在 **stop 之下、rt 之上**——所以 DL 任务**压过所有普通优先级任务和 RT 任务**。

### 1.2 关键函数地图（`kernel/sched/deadline.c`）

| 行号 | 函数 | 职责 |
|---|---|---|
| 207 | `__dl_overflow` | **准入判断**：加进来会不会超带宽 |
| 799 | `replenish_dl_entity` | 预算补充（新周期） |
| 920 | `dl_entity_overflow` | CBS 唤醒规则（密度是否超） |
| 1062 | `start_dl_timer` | 起补充定时器 |
| 1141 | `dl_server_timer` | DL server 的定时器 |
| 1211 | `dl_task_timer` | 补充定时器回调 |
| 1369 | `grub_reclaim` | **GRUB 预算折扣** |
| 1417 | `update_curr_dl_se` | **预算记账 + 节流** |
| 1579/1585 | `dl_server_update_idle` / `dl_server_update` | server 记账 |
| 1796/1820/1835 | `dl_server_start` / `_stop` / `_init` | server 生命周期 |
| 2129 | `update_curr_dl` | 更新当前 DL 任务统计 |
| 2150 | `inactive_task_timer` | **0-lag 延迟回收** |
| 2486 | `enqueue_task_dl` | 入队 |
| 2552 | `dequeue_task_dl` | 出队 |
| 2844 | `pick_task_dl` | EDF 选任务 |
| 2879 | `task_tick_dl` | tick 处理 |
| 3647 | `DEFINE_SCHED_CLASS(dl)` | 调度类方法表 |
| 3779 | `sched_dl_overflow` | `sched_setattr` 时的准入 |
| 4009 | `dl_bw_manage` | 带宽增减统一入口 |

---

## 🧱 2. 关键数据结构

### 2.1 `struct sched_dl_entity`（`include/linux/sched.h`）

内嵌在 `task_struct` 中，是 DL 任务/服务器的调度实体：

```c
struct sched_dl_entity {
	struct rb_node	rb_node;

	/* —— 原始参数：sched_setattr 时写入，之后不变 —— */
	u64	dl_runtime;   /* 每周期最大运行时间 Q */
	u64	dl_deadline;  /* 相对截止期 D */
	u64	dl_period;    /* 周期 P */
	u64	dl_bw;        /* = dl_runtime / dl_period  （利用率） */
	u64	dl_density;   /* = dl_runtime / dl_deadline（密度） */

	/* —— 运行态参数：随执行不断更新 —— */
	s64	runtime;      /* 本周期剩余预算（可能 < 0，表示超支） */
	u64	deadline;     /* 本周期绝对截止期 */

	unsigned int	flags;
	bool	dl_throttled;        /* 预算耗尽被节流 */
	bool	dl_yielded;          /* 主动让出 */
	bool	dl_non_contending;   /* 已阻塞但 0-lag 未到 */
	bool	dl_overrun;          /* 是否上报 overrun */
	/* ... dl_server / dl_defer* 等 server 相关字段 ... */
};
```

> [!NOTE] 两套参数：**原始** vs **运行态**
> `dl_runtime/dl_deadline/dl_period` 是**承诺**（不可变），`runtime/deadline` 是**可变的当前状态**。补充周期时用原始参数重置运行态——这与原理里"runtime 是上限、deadline 是绝对时刻"完全对应。

### 2.2 `struct dl_rq`（`kernel/sched/sched.h:899`）—— 三个带宽量

```c
struct dl_rq {
	struct rb_root_cached	root;        /* 按 deadline 排序的 EDF 红黑树 */
	unsigned int		dl_nr_running;
	struct { u64 curr, next; } earliest_dl;
	bool			overloaded;
	struct sched_dl_entity	*curr;
	struct rb_root_cached	pushable_dl_tasks_root;  /* 可推走（迁移）的任务 */

	u64	running_bw;   /* U_act ：活跃任务带宽和 */
	u64	this_bw;      /* U_total：归属本 rq 的任务带宽和（含非活跃） */
	u64	extra_bw;     /* U_extra：额外可回收带宽 */
	u64	max_bw;       /* U_max ：本 rq 可用的最大带宽（受 RT 限制） */
	u64	bw_ratio;     /* 可回收比例的倒数 */
};
```

- **`running_bw`**：任务**活跃**（就绪/运行/0-lag 未到）时增加，0-lag 到期时减少；
- **`this_bw`**：任务**归属**本 rq 时增加，**迁移/换策略/退出**时减少——**阻塞不改变它**；
- 于是 **非活跃带宽 `U_inact = this_bw - running_bw`**。

### 2.3 `struct dl_bw`（`kernel/sched/sched.h:348`）—— 准入账本

```c
struct dl_bw {
	raw_spinlock_t	lock;
	u64		bw;         /* 该 root domain 的带宽上限（默认 = RT 容量，约 95%） */
	u64		total_bw;   /* 已分配带宽总和 */
};
```

> 精度宏：`BW_SHIFT = 20`、`RATIO_SHIFT = 8`（`sched.h:3008`）。带宽是**定点小数**，用移位运算避免浮点。

---

## 💰 3. 预算记账与 GRUB

### 3.1 调用链

```
tick / put_prev_task_dl / dequeue_task_dl
   └─ update_curr_dl()                    (deadline.c:2129)
        └─ update_curr_dl_se(rq, dl_se, delta_exec)   (deadline.c:1417)
             ├─ scaled = dl_scaled_delta_exec(...)    // 频率/容量归一 + GRUB
             └─ dl_se->runtime -= scaled
```

`update_curr_dl` 的关键前提（`deadline.c:2135`）：

```c
if (!dl_task(donor) || !on_dl_rq(dl_se))
	return;
```

> [!TIP] 预算按"可调度时间"计，deadline 按"墙钟"算
> 源码注释明确指出：**消耗预算用可调度时间**（剔除硬中断等），**deadline 用硬墙钟**。二者口径不同，理解行为时别混。

### 3.2 GRUB：折扣公式

`dl_scaled_delta_exec()`（`deadline.c:1389`）在有 `SCHED_FLAG_RECLAIM` 时走 GRUB：

```c
if (unlikely(dl_se->flags & SCHED_FLAG_RECLAIM))
	scaled_delta_exec = grub_reclaim(delta_exec, rq, dl_se);
```

`grub_reclaim()`（`deadline.c:1369`）：

```c
static u64 grub_reclaim(u64 delta, struct rq *rq, struct sched_dl_entity *dl_se)
{
	u64 u_act;
	u64 u_inact = rq->dl.this_bw - rq->dl.running_bw;   /* Utot - Uact */

	/*
	 * 求 max{u, (u_max - u_inact - u_extra)}；
	 * 但改写为比较 u_inact + u_extra 与 u_max - u，
	 * 因为 u_inact + u_extra 可能大于 u_max，直接相减会得到负数。
	 */
	if (u_inact + rq->dl.extra_bw > rq->dl.max_bw - dl_se->dl_bw)
		u_act = dl_se->dl_bw;
	else
		u_act = rq->dl.max_bw - u_inact - rq->dl.extra_bw;

	u_act = (u_act * rq->dl.bw_ratio) >> RATIO_SHIFT;
	return (delta * u_act) >> BW_SHIFT;
}
```

对照内核文档的公式：

$$
\frac{dq}{dt} = -\frac{\max\{U_i,\ U_{\max} - U_{\text{inact}} - U_{\text{extra}}\}}{U_{\max}}
$$

- 常见的"全部已准入、无额外回收"情形下 $U_{\text{inact}} = U_{\max} - U_{\text{act}}$，且 $U_i \le U_{\text{act}}$，于是退化为 **消耗速率 $= U_{\text{act}}/U_{\max}$**；
- **只有活跃任务多时才按 1:1 扣预算**；有任务阻塞时，在跑的任务预算消耗变慢 → "借"到了空闲份额。

### 3.3 GRUB-PA：用回收带宽降频

源码注释（`deadline.c:1394`）：参与 GRUB 的任务同时实现 **GRUB-PA**——把**回收来的空闲带宽用于降低 CPU 频率**，只要还能满足截止期。当前**仅 ARM 架构实现**。

---

## 🚦 4. 节流、补充与 0-lag

### 4.1 节流（throttle）

`update_curr_dl_se()`（`deadline.c:1502`）：

```c
throttle:
	if (dl_runtime_exceeded(dl_se) || dl_se->dl_yielded) {
		trace_sched_dl_throttle_tp(dl_se, cpu_of(rq), dl_get_type(dl_se, rq));
		dl_se->dl_throttled = 1;

		/* 若用户请求，上报 overrun */
		if (dl_runtime_exceeded(dl_se) &&
		    (dl_se->flags & SCHED_FLAG_DL_OVERRUN))
			dl_se->dl_overrun = 1;

		dequeue_dl_entity(dl_se, 0);
		/* ... 起补充定时器 start_dl_timer(dl_se) ... */
	}
```

要点：

- **预算耗尽 → `dl_throttled = 1`，从 EDF 树摘除**，直到补充时刻；
- 若设置了 `SCHED_FLAG_DL_OVERRUN`，会通过 **`SIGXCPU`** 通知用户态；
- 节流**不推迟截止期**——到点仍未完成即视为错过。

### 4.2 补充（replenish）

- `start_dl_timer()`（`deadline.c:1062`）把定时器设在**截止期**（`dl_next_period()`）；
- `dl_task_timer()`（`deadline.c:1211`）到点后调用 `replenish_dl_entity()` 并**重新入队**：

```c
enqueue_task_dl(rq, p, ENQUEUE_REPLENISH);
if (dl_task(rq->donor))
	wakeup_preempt_dl(rq, p, 0);
else
	resched_curr(rq);
```

`replenish_dl_entity()`（`deadline.c:799`）的核心循环——**runtime 可能负得很深，要一直推 deadline 直到有可用预算**：

```c
while (dl_se->runtime <= 0) {
	dl_se->deadline += pi_of(dl_se)->dl_period;
	dl_se->runtime  += pi_of(dl_se)->dl_runtime;
}
```

> [!WARNING] "replenish lagged too much"
> 若推完 deadline 仍在过去，内核会打印 `sched: DL replenish lagged too much` 并**强制重置新周期**。出现这条日志，说明任务的 runtime/period 设置严重不合理（或系统长期超载）。

### 4.3 0-lag 延迟回收（`inactive_task_timer`，`deadline.c:2150`）

任务阻塞时**不立刻**从 `running_bw` 移除：

```c
no_task:
	if (dl_se->dl_non_contending == 0)
		goto unlock;

	sub_running_bw(dl_se, &rq->dl);
	dl_se->dl_non_contending = 0;
```

- 阻塞 → 置 `dl_non_contending`，起 `inactive_timer`；
- 到 **0-lag 时刻**定时器才触发，执行 `sub_running_bw()`（即 $U_{\text{act}}$ 减少）；
- 若中途醒来，则取消定时器，带宽照旧。

**为什么必须延迟**：立即回收会打破其他任务已作出的截止期承诺（对应原理篇 §4.2 的 0-lag 推导）。

---

## 🚧 5. 准入控制

### 5.1 判据（`deadline.c:207`）

```c
static inline bool
__dl_overflow(struct dl_bw *dl_b, unsigned long cap, u64 old_bw, u64 new_bw)
{
	return dl_b->bw != -1 &&
	       cap_scale(dl_b->bw, cap) < dl_b->total_bw - old_bw + new_bw;
}
```

即：**`cap_scale(bw, cap) < total_bw - old + new` 就溢出**。`bw` 默认是 RT 容量（`global_rt_runtime/global_rt_period`，默认 95%）。

### 5.2 统一入口 `dl_bw_manage()`（`deadline.c:4009`）

```c
switch (req) {
case dl_bw_req_free:
	__dl_sub(dl_b, dl_bw, dl_bw_cpus(cpu));
	break;
case dl_bw_req_alloc:
	overflow = __dl_overflow(dl_b, cap, 0, dl_bw);
	if (!overflow)
		__dl_add(dl_b, dl_bw, dl_bw_cpus(cpu));
	break;
case dl_bw_req_deactivate:
	/* CPU 下线：要留至少一个 CPU 给 DL 任务 */
	cap -= arch_scale_cpu_capacity(cpu);
	dl_server_bw = dl_server_read_bw(cpu);
	if (dl_b->total_bw - dl_server_bw > 0) {
		if (dl_bw_cpus(cpu) - 1)
			overflow = __dl_overflow(dl_b, cap, dl_server_bw, 0);
		else
			overflow = 1;
	}
	break;
}
```

对外接口：`dl_bw_alloc()` / `dl_bw_free()` / `dl_bw_deactivate()`。

### 5.3 `sched_setattr` 路径（`deadline.c:3779`）

三种情形分别处理：**新进 DL** / **DL 改参数** / **退出 DL**。

> [!NOTE] 退出时**不立刻**减 `total_bw`
> 源码注释：`switched_from_dl()` 会在正确的 **0-lag 时刻**才归还带宽。这就是原理篇强调的"销毁才归还、且要延迟"在代码里的体现。

### 5.4 参数校验（`__checkparam_dl`）

```c
if (attr->sched_deadline == 0)                      return false;
if (attr->sched_runtime < (1ULL << DL_SCALE))       return false;   /* DL_SCALE = 10 */
if (attr->sched_deadline & (1ULL << 63) ||
    attr->sched_period   & (1ULL << 63))            return false;
period = attr->sched_period ?: attr->sched_deadline;
if (period < attr->sched_deadline ||
    attr->sched_deadline < attr->sched_runtime)     return false;   /* Q <= D <= P */
if (period < min || period > max)                   return false;
```

- `sched_period == 0` → **取 `sched_deadline`**（隐式截止期）；
- 周期范围：`sysctl_sched_dl_period_min = 100 us`，`sysctl_sched_dl_period_max = 1<<22 us ≈ 4 s`；
- 最高位不能置位（用于回绕/符号判断）。

---

## 🔌 6. DL server：给 fair / ext 用的带宽服务器

这是较新且重要的一块：**DL server 让"普通调度类"也能借用 DL 的带宽隔离能力**。

- 每个 CPU 的 `rq` 里有 `fair_server`（以及可选的 `ext_server`），本身就是个 `sched_dl_entity`；
- 它**代表 fair 类**去申请一块 DL 带宽，再由它去 `pick_task_fair()` 选 fair 任务：

```c
static struct task_struct *
fair_server_pick_task(struct sched_dl_entity *dl_se, struct rq_flags *rf)
{
	return pick_task_fair(dl_se->rq, rf);
}
```

- 生命周期：CPU 上有 fair 任务时 `dl_server_start()`，队列空时 `dl_server_stop()`；
- `update_curr()` 里通过 `dl_server_update(&rq->fair_server, delta_exec)` 记账；idle 侧通过 `dl_server_update_idle()` 记账。

> [!TIP] 为什么需要它
> fair 类本身**没有**带宽上限，一个 cgroup 疯狂跑就可能拖垮整机。DL server 把 fair 任务包进一个有 CBS 预算的"服务器"里，**给普通任务也带来可回收的带宽隔离**。

---

## ⚠️ 7. 注意点（踩坑清单）

> [!WARNING] 高频坑
> 1. **`runtime ≤ deadline ≤ period` 是硬约束**，违反直接 `EINVAL`；`period = 0` 时按 `deadline` 处理。
> 2. **`runtime` 是上限不是目标**：用满 = 超支 = 节流；即使不超支，**不主动阻塞**也会被推到节流点。
> 3. **节流必然导致错过**：节流后一直停到截止期，本周期无法再跑。
> 4. **DL 任务不能 fork**：`task_fork_dl()` 是空实现，创建子进程会失败。
> 5. **DL 参数一旦设定，运行中不能改 deadline**：要变只能重建（重新 `sched_setattr`），且需重新过准入。
> 6. **带宽必须归还**：反复 create/delete 若不归还，`total_bw` 会被耗尽，之后所有新 DL 任务都 `EBUSY`。
> 7. **阻塞不等于归还**：阻塞只减 `running_bw`（且经 0-lag 延迟），`this_bw` 仍占着准入份额。
> 8. **迁移依赖 push/pull**：多核下 DL 任务通过 `push_dl_task()` / `pull_dl_task()` 迁移，`pushable_dl_tasks_root` 是专用结构。

**调试入口**：

- `/proc/sched_debug` 中的 `dl_rq` 段：`dl_nr_running`、`running_bw`、`this_bw`、`max_bw`；
- `trace_sched_dl_throttle_tp` / `trace_sched_dl_replenish_tp` 跟踪点；
- `sched_dl_period_min/max` 可经 sysctl 调整。

---

## 🧭 一句话总结

> **Linux DL = 用 EDF 红黑树选任务（`pick_task_dl`）+ 用 CBS 预算节流（`update_curr_dl_se`）+ 用 GRUB 折扣回收空闲带宽（`grub_reclaim`）+ 用 `dl_bw` 做准入（`__dl_overflow`）；阻塞要经 0-lag 才回收，销毁必须归还带宽，否则永久泄漏。**

---

## 🔗 相关

- [[principle/deadline|截止期调度原理]]：模型、EDF、CBS、GRUB 的理论
- [[linux/cfs|Linux CFS/EEVDF 实现与注意]]：fair 类与 `fair_server`
- [[linux/stop|Linux STOP 实现与注意]]、[[linux/idle|Linux IDLE 实现与注意]]
