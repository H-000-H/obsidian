---
title: Linux RT 实现与注意
tags:
  - linux-kernel
  - scheduler
  - rt
  - sched-fifo
  - sched-rr
  - rt-throttling
  - push-pull
status: Completed
created: 2026-10-07
---

# ⚡ Linux RT 实现与注意

> [!INFO] 笔记概览
> - **源码基线**：`torvalds/linux` master，**Linux 7.3.0-rc6**，commit `69f80fef3153`（2026-10-06）。文件：`kernel/sched/rt.c`（2662 行）、`kernel/sched/syscalls.c`、`kernel/sched/core.c`、`kernel/sched/sched.h`、`include/linux/sched.h`、`include/linux/sched/{rt.h,prio.h}`。
> - **定位**：把 [[principle/rt|RT 实时调度原理]] 落到 Linux 真实实现——`rt_prio_array` 位图选队、`sched_rt_entity` 组层级、`push_rt_task`/`pull_rt_task` 事件驱动迁移、`rt_bandwidth` 周期节流、以及 `RLIMIT_RTTIME` 看门狗。
> - **内容范围**：文件地图 · 数据结构 · 入队/出队 · 选择与记账 · 抢占与 RR 时间片 · 迁移（push/pull + overload 掩码）· 带宽节流 · 组调度 · 方法表逐项 · 注意点
> - **推荐路径**：§1 地图 → §2 数据结构 → §3 入队/出队 → §4 选择与记账 → §5 抢占与 tick → §6 迁移 → §7 节流 → §8 组调度 → §9 方法表 → §10 注意点

---

## 🗺️ 1. 文件地图与调度类位置

### 1.1 各文件分工

| 文件 | 职责 |
|---|---|
| `kernel/sched/rt.c` | RT **调度类**本体：位图选队、push/pull、带宽节流、RR tick |
| `kernel/sched/syscalls.c` | **用户接口**：优先级校验、`sched_setscheduler`、`sched_set_fifo*` 辅助 |
| `kernel/sched/core.c` | 调度类层级自检、`rq->rt` 初始化、`sched_set_stop_task()` |
| `kernel/sched/sched.h` | `struct rt_rq` / `rt_prio_array` / `rt_bandwidth` 定义 |
| `include/linux/sched.h` | `struct sched_rt_entity` 定义（`task_struct` 内嵌 `p->rt`） |
| `include/linux/sched/rt.h` | `rt_prio()` / `rt_task()` / `RR_TIMESLICE` |
| `include/linux/sched/prio.h` | `MAX_RT_PRIO` / `MAX_DL_PRIO` / `MAX_PRIO` |

### 1.2 层级位置

```
stop > dl > rt > fair > ext > idle
```

`core.c` 启动时用 `BUG_ON(!sched_class_above(...))` 逐对自检（与 [[linux/stop|STOP]] 笔记同一处）。RT 在 **DL 之下、fair 之上**：RT 任务能压住一切普通任务，但自己会被 DL 与 STOP 抢占。

---

## 🧩 2. 核心数据结构

### 2.1 `rt_prio_array`：位图 + 队列数组（`sched.h:311`）

```c
struct rt_prio_array {
	DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1); /* include 1 bit for delimiter */
	struct list_head queue[MAX_RT_PRIO];
};
```

- `bitmap` 有 **101** 位：0..99 对应 100 个优先级，多出的 1 位是 `sched_find_first_bit()` 的**哨兵位**（保证一定能找到终止点）；
- `queue[i]` 是优先级 `i` 的 FIFO 链表；
- 选任务 = `sched_find_first_bit(bitmap)` → `queue[idx]` 队首。

### 2.2 `sched_rt_entity`：调度实体（`include/linux/sched.h:621`）

```c
struct sched_rt_entity {
	struct list_head		run_list;
	unsigned long			timeout;
	unsigned long			watchdog_stamp;
	unsigned int			time_slice;
	unsigned short			on_rq;
	unsigned short			on_list;

	struct sched_rt_entity		*back;
#ifdef CONFIG_RT_GROUP_SCHED
	struct sched_rt_entity		*parent;
	/* rq on which this entity is (to be) queued: */
	struct rt_rq			*rt_rq;
	/* rq "owned" by this entity/group: */
	struct rt_rq			*my_q;
#endif
} __randomize_layout;
```

- 每个任务内嵌一个 `p->rt`；组调度开启时，组在每一层也有一个 `sched_rt_entity`，于是串成一棵**实体树**；
- `back` 是 `for_each_sched_rt_entity()` 的"回退链"（见 §3.2）；
- 注意：**可推列表不在这个结构里**——`pushable_tasks` 是 `task_struct` 里独立的 `plist_node`。

### 2.3 `rt_rq`：每 CPU（或每组每 CPU）的 RT 运行队列（`sched.h:864`）

```c
struct rt_rq {
	struct rt_prio_array	active;
	unsigned int		rt_nr_running;
	unsigned int		rr_nr_running;
	struct {
		int		curr; /* highest queued rt task prio */
		int		next; /* next highest */
	} highest_prio;
	bool			overloaded;
	struct plist_head	pushable_tasks;

	int			rt_queued;

#ifdef CONFIG_RT_GROUP_SCHED
	int			rt_throttled;
	u64			rt_time; /* consumed RT time, goes up in update_curr_rt */
	u64			rt_runtime; /* allotted RT time, "slice" from rt_bandwidth */
	raw_spinlock_t		rt_runtime_lock;
	unsigned int		rt_nr_boosted;
	struct rq		*rq;
#endif
#ifdef CONFIG_CGROUP_SCHED
	struct task_group	*tg;
#endif
};
```

要点：

- **`highest_prio.curr`** = 队列中最高优先级任务的 `prio`（"当前天花板"）；**`highest_prio.next`** = **可推任务**中的最高优先级（用于 pull 判定，避免取锁）；
- **`pushable_tasks`** 是按 `prio` 排序的 `plist`，只装"**可迁移**（`nr_cpus_allowed > 1`）且不在运行"的 RT 任务；
- **`rt_queued`** 表示这个 `rt_rq` 是否挂在顶层（被节流摘掉时会清零，见 §7）；
- 组调度相关的 `rt_time`/`rt_runtime`/`rt_throttled` 是**每 CPU 每组的配额**，`rt_rq->rq` 缓存顶层 rq。

### 2.4 `rt_bandwidth`：每根域的带宽（`sched.h:316`）

```c
struct rt_bandwidth {
	/* nests inside the rq lock: */
	raw_spinlock_t		rt_runtime_lock;
	ktime_t			rt_period;
	u64			rt_runtime;
	struct hrtimer		rt_period_timer;
	unsigned int		rt_period_active;
};
```

一个根域一个（`root_task_group.rt_bandwidth`），配一个 **hrtimer** 做周期性补充。

### 2.5 优先级常量（`include/linux/sched/prio.h`）

```c
#define MAX_RT_PRIO		100
#define MAX_DL_PRIO		0
#define MAX_PRIO		(MAX_RT_PRIO + NICE_WIDTH)   /* 140 */
```

`include/linux/sched/rt.h` 里的判定与时间片：

```c
static inline bool rt_prio(int prio)
{
	return unlikely(prio < MAX_RT_PRIO && prio >= MAX_DL_PRIO);
}

#define RR_TIMESLICE		(100 * HZ / 1000)   /* 默认 100ms */
```

`rt.c:10`：`int sched_rr_timeslice = RR_TIMESLICE;`

---

## 📥 3. 入队与出队

### 3.1 单实体入队：`__enqueue_rt_entity()`（`rt.c:1331`）

```c
static void __enqueue_rt_entity(struct sched_rt_entity *rt_se, unsigned int flags)
{
	struct rt_rq *rt_rq = rt_rq_of_se(rt_se);
	struct rt_prio_array *array = &rt_rq->active;
	struct rt_rq *group_rq = group_rt_rq(rt_se);
	struct list_head *queue = array->queue + rt_se_prio(rt_se);

	/*
	 * Don't enqueue the group if its throttled, or when empty.
	 */
	if (group_rq && (rt_rq_throttled(group_rq) || !group_rq->rt_nr_running)) {
		if (rt_se->on_list)
			__delist_rt_entity(rt_se, array);
		return;
	}

	if (move_entity(flags)) {
		WARN_ON_ONCE(rt_se->on_list);
		if (flags & ENQUEUE_HEAD)
			list_add(&rt_se->run_list, queue);
		else
			list_add_tail(&rt_se->run_list, queue);

		__set_bit(rt_se_prio(rt_se), array->bitmap);
		rt_se->on_list = 1;
	}
	rt_se->on_rq = 1;

	inc_rt_tasks(rt_se, rt_rq);
}
```

关键点：

- `rt_se_prio(rt_se)`（`rt.c:958`）：组调度下返回**该组 `rt_rq->highest_prio.curr`**，否则返回任务自身 `prio`。这就是原理篇 §4.1 说的"组的优先级来自组内最高成员"；
- `ENQUEUE_HEAD` 用于"让出后重排到队首"（如 `check_preempt_equal_prio` 的腾位置场景），普通入队是 `list_add_tail`；
- **被节流的组不入队**：`rt_rq_throttled(group_rq)` 为真时直接跳过。

### 3.2 层级入队/出队：栈式操作

组调度下，一个任务的入队要**沿途每一层**都更新。内核用"先全部出栈、再逐层入栈"避免优先级不一致：

```c
static void dequeue_rt_stack(struct sched_rt_entity *rt_se, unsigned int flags)
{
	struct sched_rt_entity *back = NULL;
	...
	for_each_sched_rt_entity(rt_se) {
		rt_se->back = back;
		back = rt_se;
	}
	...
	for (rt_se = back; rt_se; rt_se = rt_se->back) {
		if (on_rt_rq(rt_se))
			__dequeue_rt_entity(rt_se, flags);
	}

	dequeue_top_rt_rq(rt_rq_of_se(back), rt_nr_running);
}

static void enqueue_rt_entity(struct sched_rt_entity *rt_se, unsigned int flags)
{
	...
	dequeue_rt_stack(rt_se, flags);
	for_each_sched_rt_entity(rt_se)
		__enqueue_rt_entity(rt_se, flags);
	enqueue_top_rt_rq(&rq->rt);
}
```

> [!TIP] 为什么要"先全删再重加"
> 源码注释说得很清楚：**上层实体的优先级依赖下层**（组的优先级 = 组内最高）。所以必须**自顶向下**先全部摘掉，再自底向上重新挂上，否则中间态会算出错误的 `highest_prio`。

### 3.3 任务级入队/出队

```c
static void enqueue_task_rt(struct rq *rq, struct task_struct *p, int flags)
{
	struct sched_rt_entity *rt_se = &p->rt;

	if (flags & ENQUEUE_WAKEUP)
		rt_se->timeout = 0;

	check_schedstat_required();
	update_stats_wait_start_rt(rt_rq_of_se(rt_se), rt_se);

	enqueue_rt_entity(rt_se, flags);

	if (task_is_blocked(p))
		return;

	if (!task_current(rq, p) && p->nr_cpus_allowed > 1)
		enqueue_pushable_task(rq, p);
}

static bool dequeue_task_rt(struct rq *rq, struct task_struct *p, int flags)
{
	struct sched_rt_entity *rt_se = &p->rt;

	update_curr_rt(rq);
	dequeue_rt_entity(rt_se, flags);

	dequeue_pushable_task(rq, p);

	return true;
}
```

- 唤醒时 `timeout = 0`：重置 `RLIMIT_RTTIME` 看门狗计数（见 §5.3）；
- 只有"**非当前任务且可迁移**"才进可推列表——**正在运行的任务永远不该被推**。

### 3.4 可推列表维护（`rt.c:397` / `rt.c:413`）

```c
static void enqueue_pushable_task(struct rq *rq, struct task_struct *p)
{
	plist_del(&p->pushable_tasks, &rq->rt.pushable_tasks);
	plist_node_init(&p->pushable_tasks, p->prio);
	plist_add(&p->pushable_tasks, &rq->rt.pushable_tasks);

	/* Update the highest prio pushable task */
	if (p->prio < rq->rt.highest_prio.next)
		rq->rt.highest_prio.next = p->prio;

	if (!rq->rt.overloaded) {
		rt_set_overload(rq);
		rq->rt.overloaded = 1;
	}
}
```

`dequeue_pushable_task()` 是对称的：列表空了就把 `highest_prio.next` 复位为 `MAX_RT_PRIO-1`，并 `rt_clear_overload()`。

> [!NOTE] `overloaded` 的真实含义
> `overloaded` **不是**"RT 任务数 > 1"，而是"**这个 rq 上有可推任务**"。一旦置位，该 CPU 会登记到**根域的 `rto_mask`**（`rt_set_overload()`，`rt.c:344`），后续 pull 才有机会发现它。

---

## 🎯 4. 选择与记账

### 4.1 选最高优先级实体

```c
static struct sched_rt_entity *pick_next_rt_entity(struct rt_rq *rt_rq)
{
	struct rt_prio_array *array = &rt_rq->active;
	struct sched_rt_entity *next = NULL;
	struct list_head *queue;
	int idx;

	idx = sched_find_first_bit(array->bitmap);
	BUG_ON(idx >= MAX_RT_PRIO);

	queue = array->queue + idx;
	if (WARN_ON_ONCE(list_empty(queue)))
		return NULL;
	next = list_entry(queue->next, struct sched_rt_entity, run_list);

	return next;
}
```

`_pick_next_task_rt()` 在组调度下**逐层下降**：

```c
static struct task_struct *_pick_next_task_rt(struct rq *rq)
{
	struct sched_rt_entity *rt_se;
	struct rt_rq *rt_rq  = &rq->rt;

	do {
		rt_se = pick_next_rt_entity(rt_rq);
		if (unlikely(!rt_se))
			return NULL;
		rt_rq = group_rt_rq(rt_se);
	} while (rt_rq);

	return rt_task_of(rt_se);
}
```

`pick_task_rt()`（`rt.c:1716`）先看 `sched_rt_runnable(rq)`，再调 `_pick_next_task_rt()`。

### 4.2 `update_curr_rt()`：RT 的"记账"（`rt.c:974`）

```c
static void update_curr_rt(struct rq *rq)
{
	struct task_struct *donor = rq->donor;
	s64 delta_exec;

	if (donor->sched_class != &rt_sched_class)
		return;

	delta_exec = update_curr_common(rq);
	if (unlikely(delta_exec <= 0))
		return;

#ifdef CONFIG_RT_GROUP_SCHED
	struct sched_rt_entity *rt_se = &donor->rt;

	if (!rt_bandwidth_enabled())
		return;

	for_each_sched_rt_entity(rt_se) {
		struct rt_rq *rt_rq = rt_rq_of_se(rt_se);
		int exceeded;

		if (sched_rt_runtime(rt_rq) != RUNTIME_INF) {
			raw_spin_lock(&rt_rq->rt_runtime_lock);
			rt_rq->rt_time += delta_exec;
			exceeded = sched_rt_runtime_exceeded(rt_rq);
			if (exceeded)
				resched_curr(rq);
			raw_spin_unlock(&rt_rq->rt_runtime_lock);
			if (exceeded)
				do_start_rt_bandwidth(sched_rt_bandwidth(rt_rq));
		}
	}
#endif
}
```

> [!NOTE] RT 没有 vruntime
> 与 [[linux/cfs|fair 类]]不同，RT **不记 vruntime、不做公平排序**。`update_curr_rt()` 唯一做的事是**累加带宽消耗**并检查是否超配额。这是"静态优先级"在代码上的直接体现。

`set_next_task_rt()`（`rt.c:1657`）/`put_prev_task_rt()`（`rt.c:1728`）负责：记录 `exec_start`、把**运行中的任务从可推列表摘掉**（"running task is never eligible for pushing"）、更新 `rt_rq` 负载均值（`update_rt_rq_load_avg`，仅用于统计/PELT，不参与 RT 决策）。

---

## ⚔️ 5. 抢占与 tick

### 5.1 唤醒抢占：`wakeup_preempt_rt()`（`rt.c:1625`）

```c
static void wakeup_preempt_rt(struct rq *rq, struct task_struct *p, int flags)
{
	struct task_struct *donor = rq->donor;

	if (p->sched_class != &rt_sched_class ||
	    donor->sched_class != &rt_sched_class)
		return;

	if (p->prio < donor->prio) {
		resched_curr(rq);
		return;
	}

	/*
	 * If the newly woken task is of equal priority ... check to see if
	 * current can readily move to a different cpu.
	 */
	if (p->prio == donor->prio && !test_tsk_need_resched(rq->curr))
		check_preempt_equal_prio(rq, p);
}
```

- **严格高于** → `resched_curr()`；
- **同优先级** → 交给 `check_preempt_equal_prio()`：若新任务不可迁移而当前任务可迁移，则把当前任务**重排到队首并标记重调度**，让 push 逻辑把它挪走，给不可迁移任务腾位置；
- 非 RT 任务之间的比较**不在这里**（`p->sched_class != rt` 直接返回）。

### 5.2 RR 时间片：`task_tick_rt()`（`rt.c:2541`）

```c
static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)
{
	struct sched_rt_entity *rt_se = &p->rt;

	update_curr_rt(rq);
	update_rt_rq_load_avg(rq_clock_pelt(rq), rq, 1);

	watchdog(rq, p);

	/*
	 * RR tasks need a special form of time-slice management.
	 * FIFO tasks have no timeslices.
	 */
	if (p->policy != SCHED_RR)
		return;

	if (--p->rt.time_slice)
		return;

	p->rt.time_slice = sched_rr_timeslice;

	/*
	 * Requeue to the end of queue if we (and all of our ancestors) are not
	 * the only element on the queue
	 */
	for_each_sched_rt_entity(rt_se) {
		if (rt_se->run_list.prev != rt_se->run_list.next) {
			requeue_task_rt(rq, p, 0);
			resched_curr(rq);
			return;
		}
	}
}
```

> [!TIP] "只有我一个才不用轮转"
> 判据是 `rt_se->run_list.prev != rt_se->run_list.next`——即该优先级队列里**不止一个**元素。若整条链上每层都只有它一个，轮转没有意义，直接续跑（省一次上下文切换）。

`get_rr_interval_rt()`（`rt.c:2575`）对 FIFO 返回 **0**、对 RR 返回 `sched_rr_timeslice`；这个值被 `sched_rr_get_interval()` 导出到用户态。

### 5.3 看门狗：`RLIMIT_RTTIME`（`rt.c:2506`）

```c
static void watchdog(struct rq *rq, struct task_struct *p)
{
	unsigned long soft, hard;

	soft = task_rlimit(p, RLIMIT_RTTIME);
	hard = task_rlimit_max(p, RLIMIT_RTTIME);

	if (soft != RLIM_INFINITY) {
		unsigned long next;

		if (p->rt.watchdog_stamp != jiffies) {
			p->rt.timeout++;
			p->rt.watchdog_stamp = jiffies;
		}

		next = DIV_ROUND_UP(min(soft, hard), USEC_PER_SEC/HZ);
		if (p->rt.timeout > next)
			posix_cputimers_rt_watchdog(&p->posix_cputimers,
						    p->se.sum_exec_runtime);
	}
}
```

对 FIFO 任务，tick **唯一**的作用就是这个看门狗：连续占用 CPU 超过 `RLIMIT_RTTIME` 就发信号（默认 `SIGXCPU`）。它是"**防止 RT 任务失控**"的最后一道保险（比节流更早、更细粒度）。

---

## 🔀 6. 迁移：push / pull

RT 的迁移是**事件驱动**的，通过 `balance_callback` 机制在安全点执行：

```c
static inline void rt_queue_push_tasks(struct rq *rq)
{
	if (!has_pushable_tasks(rq))
		return;
	queue_balance_callback(rq, &per_cpu(rt_push_head, rq->cpu), push_rt_tasks);
}

static inline void rt_queue_pull_task(struct rq *rq)
{
	queue_balance_callback(rq, &per_cpu(rt_pull_head, rq->cpu), pull_rt_task);
}
```

触发点：

| 时机 | 调用 |
|---|---|
| `set_next_task_rt()` | `rt_queue_push_tasks(rq)`（开始跑 RT 任务时，把别的挤出去） |
| `switched_to_rt()`（`rt.c:2443`） | 变为 RT 且 `overloaded` → push；若优先级高于当前 → `resched_curr` |
| `prio_changed_rt()`（`rt.c:2471`） | 运行中优先级**下降** → `rt_queue_pull_task` |
| `balance_rt()`（`rt.c:1599`） | 本 rq 无 RT 任务但 `need_pull_rt_task()` → `pull_rt_task` |

### 6.1 `push_rt_task()`（`rt.c:1960`）

```c
static int push_rt_task(struct rq *rq, bool pull)
{
	struct task_struct *next_task;
	struct rq *lowest_rq;
	int ret = 0;

	if (!rq->rt.overloaded)
		return 0;

	next_task = pick_next_pushable_task(rq);
	if (!next_task)
		return 0;
retry:
	/* 若可推任务反而比当前优先级高 → 直接重调度，不推 */
	if (unlikely(next_task->prio < rq->donor->prio)) {
		resched_curr(rq);
		return 0;
	}
	...
	lowest_rq = find_lock_lowest_rq(next_task, rq);
	if (!lowest_rq) { ... }
	move_queued_task_locked(rq, lowest_rq, next_task);
	resched_curr(lowest_rq);
	ret = 1;
	...
}
```

要点：

- 被推的是**可推列表中优先级最高**的那个（`pick_next_pushable_task()`，`rt.c:1866`，会跳过 `migration_disabled` 的任务）；
- 目标 = `find_lock_lowest_rq()`（`rt.c:1897`）：在根域内找 `highest_prio.curr` **数值最大**（优先级最低）的 CPU，并且要求 `lowest_rq->rt.highest_prio.curr > task->prio`（目标确实能容纳）；最多重试 `RT_MAX_TRIES` 次，每次都会 `double_lock_balance` 复核；
- 特殊分支：若可推任务**不可迁移**（`is_migration_disabled`），则反过来把**当前任务**推走——通过 `stop_one_cpu_nowait(rq->cpu, push_cpu_stop, ...)` 让停机线程帮忙搬运；
- `push_rt_tasks()` 循环调用直到不再有可推任务。

### 6.2 `pull_rt_task()`（`rt.c:2261`）

```c
static void pull_rt_task(struct rq *this_rq)
{
	int this_cpu = this_rq->cpu, cpu;
	int rt_overload_count = rt_overloaded(this_rq);

	if (likely(!rt_overload_count))
		return;

	smp_rmb();   /* 配对 rt_set_overload() 的 smp_wmb() */

	/* If we are the only overloaded CPU do nothing */
	if (rt_overload_count == 1 &&
	    cpumask_test_cpu(this_rq->cpu, this_rq->rd->rto_mask))
		return;
	...
	for_each_cpu(cpu, this_rq->rd->rto_mask) {
		...
		if (src_rq->rt.highest_prio.next >= this_rq->rt.highest_prio.curr)
			continue;              /* 源上可推任务不够高，跳过（免取锁） */
		...
		p = pick_highest_pushable_task(src_rq, this_cpu);
		if (p && (p->prio < this_rq->rt.highest_prio.curr)) { ... }
	}
}
```

- 只扫描 **`rto_mask`**（被标记 overloaded 的 CPU），并用 `highest_prio.next` 做**免锁预筛**；
- 只拉"**优先级高于本 rq 即将运行的任务**"的 RT 任务；
- `smp_rmb()` 与 `rt_set_overload()` 里的 `smp_wmb()` 配对：保证"看到 overloaded 就一定能看到掩码位"。

> [!NOTE] `HAVE_RT_PUSH_IPI`（`rt.c:2085`）
> 若开启 `CONFIG_IRQ_WORK` 且 SMP，pull 可以改为**发 IPI 让源 CPU 自己 push**（`tell_cpu_to_push()`），避免跨 CPU 抢锁——高核数下更省。

---

## 🚧 7. 带宽节流：`rt_bandwidth`

### 7.1 默认值与开关

`rt.c:18` / `rt.c:24`：

```c
int sysctl_sched_rt_period = 1000000;    /* 1s */
int sysctl_sched_rt_runtime = 1000000;   /* 1s */
```

> [!WARNING] 本基线默认"不节流"
> `sched_rt_runtime_exceeded()`（`rt.c:863`）第一道判据是：
> ```c
> if (runtime >= sched_rt_period(rt_rq))
> 	return 0;
> ```
> 而默认 `runtime == period == 1s`，所以**默认不产生任何节流**。历史版本（≤ 6.x）默认 `runtime = 950000`（保留 5%）。
>
> 写 `sched_rt_runtime_us = -1` → `rt_bandwidth_enabled()`（`sched.h:853`，判据 `>= 0`）返回 false → **完全不限制**。

### 7.2 超限判定：`sched_rt_runtime_exceeded()`（`rt.c:863`）

```c
{
	u64 runtime = sched_rt_runtime(rt_rq);

	if (rt_rq->rt_throttled)
		return rt_rq_throttled(rt_rq);

	if (runtime >= sched_rt_period(rt_rq))
		return 0;

	balance_runtime(rt_rq);                 /* 先尝试借配额 */
	runtime = sched_rt_runtime(rt_rq);
	if (runtime == RUNTIME_INF)
		return 0;

	if (rt_rq->rt_time > runtime) {
		struct rt_bandwidth *rt_b = sched_rt_bandwidth(rt_rq);

		if (likely(rt_b->rt_runtime)) {
			rt_rq->rt_throttled = 1;
			printk_deferred_once("sched: RT throttling activated\n");
		} else {
			rt_rq->rt_time = 0;
		}

		if (rt_rq_throttled(rt_rq)) {
			sched_rt_rq_dequeue(rt_rq);   /* 从顶层摘掉 */
			return 1;
		}
	}
	return 0;
}
```

- 超限 → `rt_throttled = 1` → `sched_rt_rq_dequeue()` 把 `rt_rq` **从顶层摘掉**（`rt_queued = 0`），RT 任务立即不再参与调度；
- 只打印**一次** "RT throttling activated"（`printk_deferred_once`）。

### 7.3 周期补充：`do_sched_rt_period_timer()`（`rt.c:778`）

由 `sched_rt_period_timer()`（`rt.c:101`，hrtimer 回调）调用：

- 遍历 `span`（根组用 `cpu_online_mask`，否则用 `sched_rt_period_mask()`）；
- 每个 rq：`rt_time -= min(rt_time, overrun * runtime)`（按错过的周期数扣减）；
- 若原本 throttled 且扣完 `rt_time < runtime` → `rt_throttled = 0` 并 **`sched_rt_rq_enqueue()`**（重新挂回顶层）；
- 返回"是否 idle"，用于决定 hrtimer 是否继续武装。

### 7.4 配额借用：`do_balance_runtime()`（`rt.c:608`）

```c
/*
 * From runqueues with spare time, take 1/n part of their
 * spare time, but no more than our period.
 */
diff = iter->rt_runtime - iter->rt_time;
if (diff > 0) {
	diff = div_u64((u64)diff, weight);
	if (rt_rq->rt_runtime + diff > rt_period)
		diff = rt_period - rt_rq->rt_runtime;
	iter->rt_runtime -= diff;
	rt_rq->rt_runtime += diff;
	...
}
```

- 由 `RT_RUNTIME_SHARE` sched feature 控制是否启用（`do_sched_rt_period_timer()` 里 `!sched_feat(RT_RUNTIME_SHARE)` 时会先把每 rq 的 `rt_runtime` 重置为全局值，即**不共享**）；
- 借用上限是**本周期**（`rt_period`）；
- `RUNTIME_INF` 的 rq 不参与借出。

`__disable_runtime()`（`rt.c:660`）负责"**收回借出的配额**"（CPU 下线/热插拔路径）。

### 7.5 其它相关 API

| 符号 | 位置 | 作用 |
|---|---|---|
| `sched_rt_runtime(rt_rq)` | `rt.c:481` | 取该 rq 的配额 |
| `sched_rt_period(rt_rq)` | `rt.c:486` | 取该 rq 的周期 |
| `sched_rt_bandwidth(rt_rq)` | `rt.c:592` | 取所属 `rt_bandwidth` |
| `sched_rt_bandwidth_account(rt_rq)` | `rt.c:597` | 判断"是否还有配额"（hrtimer 活跃 或 `rt_time < rt_runtime`） |
| `init_rt_bandwidth()` | `rt.c:125` | 初始化 `rt_bandwidth` 与 hrtimer |
| `do_start_rt_bandwidth()` | `rt.c:136` | 启动/续期 hrtimer |

---

## 🌲 8. 组调度：`CONFIG_RT_GROUP_SCHED`

- 每个 `task_group` 有 `rt_se[]` / `rt_rq[]`（每 CPU 一个）和一个 `rt_bandwidth`（`sched.h:501`）；
- `init_tg_rt_entry()` 建立层级；`sched_group_set_rt_runtime()` / `sched_group_set_rt_period()` 允许按 cgroup 配 RT 配额；
- 组实体的优先级 = 组内最高优先级（`rt_se_prio()`）；
- 组被节流时，其 `rt_se` **不入父层队列**（§3.1），于是整组从父层"消失"。

> [!NOTE] 未开启组调度时的退化
> `rt.c:906` 起的 `#else` 分支把 `group_rt_rq()` 定义成 `NULL`、`for_each_sched_rt_entity()` 变成"只迭代一次"、`rt_rq_throttled()` 恒为 `false`——**同一份代码**，靠宏在两种配置下都成立。

---

## 🧩 9. `sched_class` 方法表逐项（`rt.c:2602`）

```c
DEFINE_SCHED_CLASS(rt) = {
	.enqueue_task		= enqueue_task_rt,
	.dequeue_task		= dequeue_task_rt,
	.yield_task		= yield_task_rt,

	.wakeup_preempt		= wakeup_preempt_rt,

	.pick_task		= pick_task_rt,
	.put_prev_task		= put_prev_task_rt,
	.set_next_task          = set_next_task_rt,

	.balance		= balance_rt,
	.select_task_rq		= select_task_rq_rt,
	.set_cpus_allowed       = set_cpus_allowed_common,
	.rq_online              = rq_online_rt,
	.rq_offline             = rq_offline_rt,
	.task_woken		= task_woken_rt,
	.switched_from		= switched_from_rt,
	.find_lock_rq		= find_lock_lowest_rq,

	.task_tick		= task_tick_rt,

	.get_rr_interval	= get_rr_interval_rt,

	.switched_to		= switched_to_rt,
	.prio_changed		= prio_changed_rt,

	.update_curr		= update_curr_rt,

#ifdef CONFIG_SCHED_CORE
	.task_is_throttled	= task_is_throttled_rt,
#endif

#ifdef CONFIG_UCLAMP_TASK
	.uclamp_enabled		= 1,
#endif
};
```

| 方法 | 实现 | 说明 |
|---|---|---|
| `enqueue/dequeue_task` | `rt.c:1436` / `1455` | 组层级入队/出队 + 可推列表维护 |
| `yield_task` | `rt.c:1496` | `requeue_task_rt(rq, rq->donor, 0)` → **排到同级队尾** |
| `wakeup_preempt` | `rt.c:1625` | 严格高优先级抢占；同级交给 `check_preempt_equal_prio` |
| `pick_task` | `rt.c:1716` | `sched_find_first_bit` + 逐层下降 |
| `put_prev_task` | `rt.c:1728` | 记账；把仍在队列的前任务重新纳入可推列表 |
| `set_next_task` | `rt.c:1657` | 摘掉可推资格 + `rt_queue_push_tasks` |
| `balance` | `rt.c:1599` | 只做 pull 机会判定，返回 `sched_*_runnable()` |
| `select_task_rq` | `rt.c:1504` | 唤醒/ fork 时才尝试选更合适的 CPU；否则保持原 CPU |
| `find_lock_rq` | `rt.c:1897` | 暴露给 `push_rt_task` 用（"找并锁住最低优先级 CPU"） |
| `task_tick` | `rt.c:2541` | RR 时间片 + 看门狗 |
| `get_rr_interval` | `rt.c:2575` | FIFO→0，RR→`sched_rr_timeslice` |
| `switched_to` / `prio_changed` | `rt.c:2443` / `2471` | 变更优先级后触发 push/pull/重调度 |
| `update_curr` | `rt.c:974` | **只**累加带宽消耗 |
| `task_is_throttled` | `rt.c:2587` | core-sched 用：查询该任务所在 rq 是否被节流 |

> [!NOTE] RT 没有 `.task_fork` / `.task_dead`
> 与 fair 类不同，RT 不在 `sched_class` 里做 fork/dead 钩子；相关状态在 `core.c` 的通用路径里初始化。

---

## 🔌 10. 用户接口与优先级换算

### 10.1 优先级映射（`syscalls.c:19`）

```c
static inline int __normal_prio(int policy, int rt_prio, int nice)
{
	int prio;

	if (dl_policy(policy))
		prio = MAX_DL_PRIO - 1;
	else if (rt_policy(policy))
		prio = MAX_RT_PRIO - 1 - rt_prio;   /* 99 - rt_priority */
	else
		prio = NICE_TO_PRIO(nice);

	return prio;
}
```

### 10.2 校验（`syscalls.c:525`）

```c
/*
 * Valid priorities for SCHED_FIFO and SCHED_RR are
 * 1..MAX_RT_PRIO-1, valid priority for SCHED_NORMAL,
 * ...
 */
if (attr->sched_priority > MAX_RT_PRIO-1)
	return -EINVAL;
if ((dl_policy(policy) && !__checkparam_dl(attr)) ||
    (rt_policy(policy) != (attr->sched_priority != 0)))
	return -EINVAL;
```

即：RT 策略**必须**带非零 `sched_priority`（1..99），非 RT 策略**必须**为 0。

### 10.3 权限

- 需要 `CAP_SYS_NICE`，或 `RLIMIT_RTPRIO` 允许提升（`syscalls.c:445` 附近有 `attr->sched_priority > rlim_rtprio` 的判定）；
- 内核自用辅助函数（`syscalls.c:812` / `822`）：
  - `sched_set_fifo(p)` → `MAX_RT_PRIO / 2`（50）；
  - `sched_set_fifo_low(p)` → `1`。

### 10.4 `sysfs` / `procfs` 旋钮

| 路径 | 默认 | 说明 |
|---|---|---|
| `/proc/sys/kernel/sched_rt_period_us` | 1000000 | 统计周期（µs） |
| `/proc/sys/kernel/sched_rt_runtime_us` | 1000000 | 周期内配额（µs）；**-1 = 不限** |
| `/proc/sys/kernel/sched_rr_timeslice_ms` | 100 | RR 时间片（ms） |

---

## ⚠️ 11. 注意点（踩坑清单）

> [!WARNING] 高频坑
> 1. **默认不节流**：本基线 `runtime == period == 1s`，等于关闭节流。生产环境**务必显式设置** `sched_rt_runtime_us`，否则一个死循环 RT 任务能锁死系统。
> 2. **FIFO 无时间片**：`task_tick_rt()` 对 FIFO **只**做 `RLIMIT_RTTIME` 看门狗，不做时间片轮转。想让同级任务轮流跑，必须用 **SCHED_RR**。
> 3. **`yield()` 不是进度保证**：源码注释（`syscalls.c:1373` 附近）明确警告 `while (!event) yield();` 在 FIFO 下**永不成立**。
> 4. **`prio` 数字小 = 优先级高**：读代码时不要按直觉理解。
> 5. **PI 会改写有效优先级**：`rt_task(p)` 看的是**有效** `prio`，含 PI 提升；要判"原始策略"用 `rt_or_dl_task_policy()`。
> 6. **`overloaded` 的语义**：是"有可推任务"，不是"RT 任务数 > 1"；只有 `overloaded` 的 CPU 才进 `rto_mask`，pull 才会看到它。
> 7. **迁移是事件驱动的**：RT **不**参与 CFS 周期性负载均衡；如果 push/pull 的触发点没被走到，任务可能"待在原地"（这是设计，不是漏）。
> 8. **组调度下"组的优先级"会变**：组内最高优先级任务进出队会改变组在父层的优先级，因此入队必须"先全删再重加"（§3.2）。
> 9. **节流是保险丝，不是公平**：它只保证"系统还能跑"，不保证 RT 任务之间公平。
> 10. **`RUNTIME_INF`**：CPU 下线等路径会把 `rt_runtime` 设为 `RUNTIME_INF`，此时不参与借用也不被节流。

**调试入口**：

- `print_rt_rq()`（`sched.h:3441` 声明）打印每 CPU RT 队列；
- 内核日志出现 `sched: RT throttling activated` 说明触发了节流；
- `sched_debug`（`/proc/sched_debug`）里有 `rt_nr_running` / `highest_prio` / `pushable` 等字段。

---

## 🧭 一句话总结

> **Linux RT = `rt.c` 里一个"位图 + 每优先级一条链表"的 O(1) 调度类：`__enqueue_rt_entity()` 置位并挂链表，`pick_next_rt_entity()` 用 `sched_find_first_bit()` 取最高优先级；`wakeup_preempt_rt()` 做严格抢占、`task_tick_rt()` 只给 RR 递减时间片（FIFO 仅做 `RLIMIT_RTTIME` 看门狗）；迁移靠 `push_rt_task()`/`pull_rt_task()` 经 `rto_mask` 事件驱动；`rt_bandwidth` 用"周期 + 配额 + hrtimer 补充 + 跨 CPU 借用"做节流兜底——注意本基线默认 `runtime == period`，等于默认关闭节流。**

---

## 🔗 相关

- [[principle/rt|RT 实时调度原理]]：优先级模型、FIFO/RR 语义、节流与优先级反转
- [[linux/cfs|Linux CFS/EEVDF 实现与注意]]：fair 类实现（红黑树 vs 位图）
- [[linux/deadline|Linux DL 实现与注意]]、[[linux/stop|Linux STOP 实现与注意]]、[[linux/idle|Linux IDLE 实现与注意]]、[[linux/ext|Linux sched_ext 实现与注意]]