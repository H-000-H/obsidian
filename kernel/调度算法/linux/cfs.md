---
title: Linux CFS/EEVDF 实现与注意
tags:
  - linux-kernel
  - scheduler
  - cfs
  - eevdf
  - vruntime
  - lag
  - cfs-bandwidth
status: Completed
created: 2026-10-06
---

# ⚖️ Linux CFS/EEVDF 实现与注意

> [!INFO] 笔记概览
> - **源码基线**：`torvalds/linux` master，**Linux 7.3.0-rc6**，commit `22430ae5d90a`（2026-10-06）。文件：`kernel/sched/fair.c`、`kernel/sched/sched.h`、`include/linux/sched.h`。
> - **定位**：把 [[principle/cfs|CFS 完全公平调度原理]] 落到 Linux 真实实现——**6.6 起 fair 类的核心算法已是 EEVDF**，不再是"纯 vruntime 最左"。本篇给出**关键数据结构、核心函数与源码位置**，并区分"经典 CFS 残留"与"EEVDF 现状"。
> - **内容范围**：文件地图 · `sched_entity` / `cfs_rq` / `cfs_bandwidth` 数据结构 · avg_vruntime 与 lag · 虚拟截止期与增广红黑树 · `pick_eevdf` 选任务 · 记账 · 放置策略 · 抢占与保护片 · CFS 带宽限流 · 组调度与 `fair_server` · 注意点
> - **推荐路径**：§1 地图 → §2 数据结构 → §3 lag/合格性 → §4 虚拟截止期与红黑树 → §5 选任务 → §6 记账 → §7 放置 → §8 抢占 → §9 时间片 → §10 带宽 → §11 注意点

---

## 🗺️ 1. 文件地图与调度类位置

### 1.1 位置

fair 类在层级中位于 **rt 之下、ext/idle 之上**（顺序由 `include/asm-generic/vmlinux.lds.h` 的 `SCHED_DATA` 段决定，`core.c:8987` 起有 `BUG_ON` 自检）：

```
stop > dl > rt > fair > ext > idle
```

普通进程（`SCHED_NORMAL` / `SCHED_BATCH`）与 `SCHED_IDLE` **策略**的任务都落在 fair 类。

### 1.2 关键函数地图（`kernel/sched/fair.c`）

| 行号 | 函数 | 职责 |
|---|---|---|
| 267/294 | `__calc_delta` | 定点换算：`delta * weight / lw->weight` |
| 303 | `calc_delta_fair` | **虚拟时间增量**：`delta * NICE_0_LOAD / weight` |
| 558 | `entity_before` | 排序比较：按 `deadline` 比大小 |
| 749 | `avg_vruntime` | 计算加权平均虚拟时间 $V$（即 `zero_vruntime`） |
| 814 | `ineligible_vruntime` | 不合格实体的参考 vruntime |
| 862 | `entity_lag` | **lag** 计算并 clamp |
| 888 | `update_entity_lag` | 出队/延迟出队时更新 lag |
| 922 | `vruntime_eligible` | 整数化的合格判断（免除法） |
| 967 | `entity_eligible` | 合格判断封装 |
| 987 | `cfs_rq_max_slice` | 队列内最大片（增广字段） |
| 1045 | `min_vruntime_update` | 增广红黑树字段维护 |
| 1076/1090 | `__enqueue_entity` / `__dequeue_entity` | 增广红黑树插入/删除 |
| 1128 | `set_protect_slice` | 设置"保护片"上界 `vprot` |
| 1187 | `pick_eevdf` | **EEVDF 选任务** |
| 1289 | `update_deadline` | 片耗尽后推进虚拟截止期 |
| 1406 | `update_se` | 累计真实执行时间 |
| 2223 | `update_curr` | **记账主函数** |
| 6383 | `place_entity` | **入队放置**（vruntime / deadline） |
| 6769 | `entity_tick` | tick 处理 |
| 7202 | `throttle_cfs_rq` | 带宽节流 |
| 7267 | `unthrottle_cfs_rq` | 解除节流 |
| 10126 | `wakeup_preempt_fair` | **唤醒抢占判断** |
| 10270 | `pick_task_fair` | fair 类 pick 入口 |
| 15332 | `task_tick_fair` | tick 入口 |
| 15847 | `DEFINE_SCHED_CLASS(fair)` | 调度类方法表 |

---

## 🧱 2. 关键数据结构

### 2.1 `struct sched_entity`（`include/linux/sched.h:572`）

CFS 的调度实体。**EEVDF 相关字段已在其中**：

```c
struct sched_entity {
	/* For load-balancing: */
	struct load_weight	load;
	struct load_weight	h_load;
	struct rb_node		run_node;
	u64			deadline;      /* 虚拟截止期 vd_i（排序键！） */
	u64			min_vruntime;  /* 增广：子树最小 vruntime */
	u64			min_slice;     /* 增广：子树最小 slice */
	u64			max_slice;     /* 增广：子树最大 slice */

	struct list_head	group_node;
	unsigned char		on_rq;
	unsigned char		sched_delayed; /* 延迟出队中 */
	unsigned char		rel_deadline;  /* deadline 是相对值，放置时再加 */
	unsigned char		custom_slice;

	u64			exec_start;
	u64			sum_exec_runtime;
	u64			prev_sum_exec_runtime;
	u64			vruntime;
	s64			vlag;          /* 近似虚拟 lag */
	u64			vprot;         /* 受保护的 vruntime 上界 */
	u64			slice;         /* 本次请求 r_i */
	/* ... */
	struct sched_avg	avg;
};
```

> [!NOTE] 与经典 CFS 的差别
> 经典 CFS 只有 `vruntime` 一个键；EEVDF 追加了 `deadline`（虚拟截止期，**红黑树真正的排序键**）、`vlag`（lag）、`vprot`（保护片）、`slice`（请求大小）。`min_vruntime/min_slice/max_slice` 是**增广红黑树**用的子树聚合量。

### 2.2 `struct cfs_rq`（`kernel/sched/sched.h:681`）

```c
struct cfs_rq {
	struct load_weight	load;
	unsigned int		nr_queued;
	unsigned int		h_nr_queued;    /* 含组，SCHED_{NORMAL,BATCH,IDLE} */
	unsigned int		h_nr_runnable;
	unsigned int		h_nr_idle;      /* SCHED_IDLE */

	s64			sum_w_vruntime; /* Σ(v_i - v0)*w_i，avg_vruntime 的原料 */
	u64			sum_weight;     /* Σ w_i */
	u64			zero_vruntime;  /* 基准点 v0，即 V 的整数表示 */
	unsigned int		sum_shift;

	struct rb_root_cached	tasks_timeline; /* 以 deadline 排序的增广红黑树 */
	struct sched_entity	*curr;          /* 当前正在跑的实体 */
	struct sched_entity	*next;          /* buddy 提示 */

	struct sched_avg	avg;
	/* ... */
	u64			runtime_remaining; /* CFS 带宽剩余额度 */
	int			throttled;
	/* ... */
};
```

- **`zero_vruntime`** 是 CFS/EEVDF 的"虚拟时间基准"，`avg_vruntime()` 每次计算后把它推进到当前加权平均 $V$；
- `sum_w_vruntime` 与 `sum_weight` 一起支撑"免除法"的合格性判断（§3.3）。

### 2.3 `struct cfs_bandwidth`（`kernel/sched/sched.h:451`）

```c
struct cfs_bandwidth {
	raw_spinlock_t	lock;
	ktime_t		period;            /* 周期 P */
	u64		quota;             /* 每周期配额 Q */
	u64		runtime;           /* 全局剩余额度 */
	u64		burst;             /* 突发额度上限 */
	s64		hierarchical_quota;
	u8		idle;
	struct hrtimer	period_timer;
	struct hrtimer	slack_timer;
	struct list_head throttled_cfs_rq; /* 被节流的队列链表 */
	/* 统计：nr_periods / nr_throttled / throttled_time ... */
};
```

> 精度宏：`BW_SHIFT = 20`、`RATIO_SHIFT = 8`（`sched.h`）。带宽是**定点小数**，移位代替浮点。

---

## 🧮 3. lag 与合格性（EEVDF 的地基）

### 3.1 加权平均虚拟时间 `avg_vruntime`（`fair.c:749`）

EEVDF 的 $V$ 由 `avg_vruntime()` 求出，结果缓存在 `cfs_rq->zero_vruntime`：

$$
V = \frac{\sum_j v_j w_j}{\sum_j w_j} = \frac{\sum_j (v_j - v_0) w_j}{\sum_j w_j} + v_0
$$

源码用 `sum_w_vruntime = Σ(v_j - v0)·w_j` 和 `sum_weight = Σw_j` 两个累加量避免直接遍历：

```c
s64 runtime = cfs_rq->sum_w_vruntime;
if (curr) {
	unsigned long w = avg_vruntime_weight(cfs_rq, curr->h_load.weight);
	runtime += entity_key(cfs_rq, curr) * w;
	weight += w;
}
/* sign flips effective floor / ceiling */
if (runtime < 0)
	runtime -= (weight - 1);
delta = div64_long(runtime, weight);
update_zero_vruntime(cfs_rq, delta);
return cfs_rq->zero_vruntime;
```

> [!TIP] 为什么要有 `zero_vruntime` 基准点
> 虚拟时间会随运行**单调增长到溢出**。引入基准点 $v_0$ 后，树里只存"相对 $v_0$ 的偏移"，增量小、不溢出，且比较运算天然带符号。

### 3.2 lag 与 clamp（`fair.c:862`）

$$
\text{lag}_i = S_i - s_i = w_i\,(V - v_i)
\;\Longleftrightarrow\;
\text{vlag}_i = V - v_i
$$

源码只跟踪**虚拟 lag**（省掉 $w_i$）：

```c
static s64 entity_lag(struct cfs_rq *cfs_rq, struct sched_entity *se, u64 avruntime)
{
	u64 max_slice = cfs_rq_max_slice(cfs_rq) + TICK_NSEC;
	s64 vlag, limit;

	vlag = avruntime - se->vruntime;
	limit = calc_delta_fair(max_slice, se);

	return clamp(vlag, -limit, limit);   /* 稳态界：-r_max < lag < max(r_max, q) */
}
```

> [!WARNING] lag 为什么要 clamp
> $V$ 是加权平均，任务增删/改权重都会让 $V$ **跳动**，可能算出比实际大得多的 lag。源码把 lag 限制在"两倍片长（下限 TICK_NSEC）"内，对应原理篇的稳态界。

### 3.3 合格性：用整数比较代替除法（`fair.c:922`）

$$
\text{lag}_i \ge 0 \iff V \ge v_i
\iff \sum_j (v_j - v_0) w_j \ge (v_i - v_0)\sum_j w_j
$$

`vruntime_eligible()` 正是这样**两边同乘 $\sum w_j$、避免浮点除法**：

```c
s64 key = vruntime_op(vruntime, "-", cfs_rq->zero_vruntime);
...
#ifdef CONFIG_64BIT
#ifdef CONFIG_ARCH_SUPPORTS_INT128
	return avg >= (__int128)key * load;   /* 用 128 位防溢出 */
#else
	if (check_mul_overflow(key, load, &rhs))
		return key <= 0;                  /* 溢出时看符号即可判断 */
	return avg >= rhs;
#endif
```

`entity_eligible()`（`fair.c:967`）是它的封装，供 `pick_eevdf` 反复调用。

---

## 🌲 4. 虚拟截止期与增广红黑树

### 4.1 虚拟截止期

$$
vd_i = v_i + \frac{r_i}{w_i}
$$

源码里 $r_i$ 就是 `se->slice`，`vd_i` 就是 `se->deadline`。片耗尽时由 `update_deadline()`（`fair.c:1289`）推进：

```c
static bool update_deadline(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	if (vruntime_cmp(se->vruntime, "<", se->deadline))
		return false;

	if (!se->custom_slice)
		se->slice = sysctl_sched_base_slice;   /* 默认 0.7ms */

	/* EEVDF: vd_i = ve_i + r_i / w_i */
	se->deadline = se->vruntime + calc_delta_fair(se->slice, se);
	avg_vruntime(cfs_rq);

	return true;   /* 片用完了，请求重新调度 */
}
```

### 4.2 排序键是 deadline，不是 vruntime

```c
static inline bool entity_before(const struct sched_entity *a,
				 const struct sched_entity *b)
{
	/* Tiebreak on vruntime seems unnecessary since it can hardly happen. */
	return vruntime_cmp(a->deadline, "<", b->deadline);
}
```

所以 `tasks_timeline` 这棵红黑树**按虚拟截止期排序**，最左节点 = 最早截止期——这正是 EEVDF 名字里的 "Earliest ... First"。

### 4.3 增广字段：让"找合格实体"变成 $O(\log n)$

如果只按 deadline 排序，找"最早的**合格**实体"就得退化成线性扫描。EEVDF 的做法是在每个节点**增广**三个子树聚合量：

| 增广字段 | 含义 | 用途 |
|---|---|---|
| `min_vruntime` | 子树最小 vruntime | 判断左子树里是否还有合格实体 |
| `min_slice` | 子树最小 slice | `RUN_TO_PARITY` 时取最短片 |
| `max_slice` | 子树最大 slice | `entity_lag` 的 clamp 上界 |

维护逻辑在 `min_vruntime_update()`（`fair.c:1045`）：

```c
se->min_vruntime = se->vruntime;
__min_vruntime_update(se, node->rb_right);
__min_vruntime_update(se, node->rb_left);
/* min_slice / max_slice 同理 */
```

插入/删除用 `RB_DECLARE_CALLBACKS_MULTI` 生成的 `min_vruntime_cb`：

```c
static void __enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	sum_w_vruntime_add(cfs_rq, se);
	se->min_vruntime = se->vruntime;
	se->min_slice = se->slice;
	se->max_slice = se->slice;

	rb_add_augmented_cached(&se->run_node, &cfs_rq->tasks_timeline,
				__entity_less, &min_vruntime_cb);
}
```

---

## 🎯 5. 选任务：`pick_eevdf`（`fair.c:1187`）

```c
static struct sched_entity *pick_eevdf(struct cfs_rq *cfs_rq, bool protect)
{
	struct rb_node *node = cfs_rq->tasks_timeline.rb_root.rb_node;
	struct sched_entity *se = __pick_first_entity(cfs_rq);
	struct sched_entity *curr = cfs_rq->curr;
	struct sched_entity *best = NULL;

	if (cfs_rq->h_nr_queued == 1)                 /* 只有一个，直接返回 */
		return curr && curr->on_rq ? curr : se;

	if (sched_feat(PICK_BUDDY) && protect &&
	    cfs_rq->next && entity_eligible(cfs_rq, cfs_rq->next))
		return cfs_rq->next;                       /* buddy 提示（只影响延迟不影响公平） */

	if (curr && (!curr->on_rq || !entity_eligible(cfs_rq, curr)))
		curr = NULL;

	if (curr && protect && protect_slice(curr))     /* 保护片内不切 */
		return curr;

	if (se && entity_eligible(cfs_rq, se)) {        /* 最左（最早 deadline）合格 */
		best = se;
		goto found;
	}

	/* Heap search：沿树找最早的合格实体 */
	while (node) {
		struct rb_node *left = node->rb_left;
		if (left && vruntime_eligible(cfs_rq, __node_2_se(left)->min_vruntime)) {
			node = left;                            /* 左子树有合格实体 → 更早 */
			continue;
		}
		se = __node_2_se(node);
		if (entity_eligible(cfs_rq, se)) {
			best = se;
			break;
		}
		node = node->rb_right;                      /* 左子树与自身都不合格 → 往右 */
	}
found:
	if (!best || (curr && entity_before(curr, best)))
		best = curr;
	return best;
}
```

> [!NOTE] 三步走
> 1. **最左合格** → 直接选（最常见）；
> 2. 否则用增广字段做**堆式剪枝搜索**：左子树有合格实体就下沉，否则看本节点，再否则往右；
> 3. 最后和 `curr` 比 deadline 取更早的。
>
> 全程 $O(\log n)$，无需遍历整棵树。

---

## 💰 6. 记账：`update_curr`（`fair.c:2223`）

调用链：`tick / put_prev_task / dequeue` → `update_curr_fair` → `update_curr`。

```c
delta_exec = update_se(rq, curr);
if (unlikely(delta_exec <= 0))
	return;

account_cfs_rq_runtime(cfs_rq, delta_exec);   /* 带宽记账（若启用） */
...
curr->vruntime += calc_delta_fair(delta_exec, curr);   /* 虚拟时间推进 */
resched = update_deadline(cfs_rq, curr);               /* 片是否用完 */

dl_server_update(&rq->fair_server, delta_exec);        /* 记到 fair_server */
...
if (resched || !protect_slice(curr)) {
	resched_curr_lazy(rq);
	clear_buddies(cfs_rq, curr);
}
```

核心一行：

$$
\Delta \text{vruntime} = \text{calc\_delta\_fair}(\Delta t, se) = \Delta t \cdot \frac{\text{NICE\_0\_LOAD}}{w}
$$

`calc_delta_fair()`（`fair.c:303`）仅在 `h_load.weight != NICE_0_LOAD` 时走 `__calc_delta`，否则原样返回——**nice 0 的任务 vruntime 增量就等于真实时间**。

---

## 📥 7. 放置：`place_entity`（`fair.c:6383`）

入队（含唤醒）前决定实体的 `vruntime` 与 `deadline`。

### 7.1 保留 lag（`PLACE_LAG`）

源码注释推导了"加入一个实体后 $V$ 会被拉低、lag 会蒸发"的问题，并给出反解：

$$
vl_i = \frac{(W + w_i)\,vl'_i}{W}
$$

```c
if (sched_feat(PLACE_LAG) && nr_queued && se->vlag) {
	lag = se->vlag;
	load = cfs_rq->sum_weight;
	if (curr && curr->on_rq)
		load += avg_vruntime_weight(cfs_rq, curr->h_load.weight);
	weight = avg_vruntime_weight(cfs_rq, se->h_load.weight);

	lag *= load + weight;
	if (WARN_ON_ONCE(!load))
		load = 1;
	lag = div64_long(lag, load);
	...
}
se->vruntime = vruntime - lag;
```

> [!TIP] 这正是"睡眠补偿"的正式做法
> 经典 CFS 用 `min_vruntime` 把睡眠任务"垫高"（启发式）；EEVDF 直接把睡眠期间的 **lag** 保留下来，入队时**按比例放大**后还原。原理篇 §3.2 提到的"补丁变一等公民"就落在这里。

### 7.2 新任务只给半片（`PLACE_DEADLINE_INITIAL`）

```c
if (sched_feat(PLACE_DEADLINE_INITIAL) && (flags & ENQUEUE_INITIAL))
	vslice /= 2;                                    /* 新任务先给半片，避免一上来就抢 */

se->deadline = se->vruntime + vslice;               /* EEVDF: vd_i = ve_i + r_i/w_i */
```

### 7.3 迁移保留相对截止期（`PLACE_REL_DEADLINE`）

```c
if (sched_feat(PLACE_REL_DEADLINE) && se->rel_deadline) {
	se->deadline += se->vruntime;    /* 之前存的是相对值，落位后转成绝对 */
	se->rel_deadline = 0;
	return;
}
```

> 迁移到新 CPU 时，deadline 若原样搬过去会与目标 CPU 的虚拟时间尺度错位，于是先存**相对值**，放置时再加到新 `vruntime` 上。

---

## ⚡ 8. 抢占：`wakeup_preempt_fair`（`fair.c:10126`）

### 8.1 抢占判据（优先级从高到低）

| 判据 | 行为 | 开关 |
|---|---|---|
| 当前是 idle 实体、唤醒的是普通实体 | 直接抢占 | — |
| 唤醒实体 **slice 更短** | 走 `PREEMPT_WAKEUP_SHORT`，**越过保护片**抢占 | `PREEMPT_SHORT` |
| 唤醒实体成为 `pick_eevdf` 的 `next` | 抢占 | — |
| 唤醒实体合格但非 next | 取消当前保护片，防止大延迟 | `PREEMPT_SHORT` |

```c
if (cse_is_idle && !pse_is_idle)
	goto preempt;

/* @p 片更短且合格 → 覆盖 curr 的保护片 */
if (sched_feat(PREEMPT_SHORT) && (pse->slice < se->slice)) {
	preempt_action = PREEMPT_WAKEUP_SHORT;
	goto pick;
}
...
	if (nse == pse)          /* @p 成了最合格者，强制抢占 */
		goto preempt;
```

### 8.2 保护片（`RUN_TO_PARITY`）：`vprot`

`set_protect_slice()`（`fair.c:1128`）给当前实体设一个"至少跑到这里"的 vruntime 上界 `vprot`：

```c
static inline void set_protect_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	u64 slice = normalized_sysctl_sched_base_slice;
	u64 vprot = se->deadline;

	if (sched_feat(RUN_TO_PARITY))
		slice = cfs_rq_min_slice(cfs_rq);      /* run-to-parity：取队列最短片 */
	slice = min(slice, se->slice);
	...
	se->vprot = vprot;
}

static inline bool protect_slice(struct sched_entity *se)
{
	return vruntime_cmp(se->vruntime, "<", se->vprot);   /* 还没到上界 → 受保护 */
}
```

> [!NOTE] run-to-parity 的含义
> 开启 `RUN_TO_PARITY` 后，当前任务至少跑到"追平 0-lag 点或耗尽最短片"，期间不被普通唤醒抢占——**减少过度切换、提升吞吐**；但允许**更短片**的任务（`PREEMPT_SHORT`）插队，兼顾延迟。

### 8.3 tick 侧

`task_tick_fair()`（`fair.c:15332`）→ `entity_tick()`（`fair.c:6769`）：`update_curr` + 负载跟踪。注意 **EEVDF 下 tick 不再负责"时间片到期"判断**——片耗尽由 `update_curr` 里的 `update_deadline()` 返回值触发 `resched_curr_lazy`；开启 `HRTICK` 时改用高精度定时器对齐片边界。

---

## ⏱️ 9. 时间片与调度粒度

| 参数 | 位置 | 默认值 | 含义 |
|---|---|---|---|
| `sysctl_sched_base_slice` | `fair.c:85` | **700000 ns（0.7 ms）** | EEVDF 的基准请求大小 $r_i$ |
| `sysctl_sched_cfs_bandwidth_slice` | `fair.c:131` | 5000 µs | 带宽额度按此粒度发放 |

- **片是"请求大小"不是"配额"**：`se->slice` 默认等于 `sched_base_slice`，`custom_slice` 时可由用户/框架指定；
- 队列里片的**最小值/最大值**通过增广字段 $O(1)$ 取得（`cfs_rq_min_slice` / `cfs_rq_max_slice`），供 `vprot` 与 lag clamp 使用；
- 经典 CFS 的 `sched_latency`(6ms) / `sched_min_granularity`(0.75ms) 已不再是 fair 的核心参数——**这是最容易残留的旧知识**。

---

## 🚧 10. CFS 带宽限流（`CONFIG_CFS_BANDWIDTH`）

### 10.1 模型

给一个 task_group 设"每周期最多用多少 CPU 时间"（`cpu.max` → `quota/period`），超出即 **throttle**。

### 10.2 记账与节流

```c
static bool __account_cfs_rq_runtime(struct cfs_rq *cfs_rq, u64 delta_exec)
{
	cfs_rq->runtime_remaining -= delta_exec;      /* 先扣 */
	if (likely(cfs_rq->runtime_remaining > 0))
		return false;
	if (cfs_rq->throttled)
		return true;
	return throttle_cfs_rq(cfs_rq);               /* 扣光 → 先尝试续额度，再节流 */
}
```

`throttle_cfs_rq()`（`fair.c:7202`）会**先尝试续额度**，续不到才真正节流：

```c
if (curr && curr->on_rq)
	target_runtime = sched_cfs_bandwidth_slice();   /* 还要跑 → 再要 5ms */
if (__assign_cfs_rq_runtime(cfs_b, cfs_rq, target_runtime))
	return false;                                    /* 续到了，不节流 */
list_add_tail_rcu(&cfs_rq->throttled_list, &cfs_b->throttled_cfs_rq);
cfs_rq->throttled = 1;
```

### 10.3 补充与突发

`__refill_cfs_bandwidth_runtime()`（`fair.c:6840`）每周期把全局额度加回 `quota`，并按 `burst` 封顶：

```c
cfs_b->runtime += cfs_b->quota;
runtime = cfs_b->runtime_snap - cfs_b->runtime;
if (runtime > 0) {
	cfs_b->burst_time += runtime;
	cfs_b->nr_burst++;
}
cfs_b->runtime = min(cfs_b->runtime, cfs_b->quota + cfs_b->burst);
```

- 周期定时器 `do_sched_cfs_period_timer()`（`fair.c:7480`）负责补充 + 解除节流；
- slack 定时器 `do_sched_cfs_slack_timer()`（`fair.c:7623`）负责把**闲置额度**分给其他队列（对应原理篇 §6.2"限流与公平正交"）。

> [!NOTE] 与公平的关系
> 带宽限制的是**一组任务的总额**；组内部依旧按 EEVDF 公平。两者正交。

---

## 🏢 11. 组调度与 `fair_server`

### 11.1 task_group 递归

启用 `CONFIG_FAIR_GROUP_SCHED` 后，**任务组本身也是一个 `sched_entity`** 参与竞争：

- 先在组之间按权重公平，再在组内按权重公平，递归向下；
- `cpu.weight` 控制的就是这条链上的权重。

### 11.2 `fair_server`：把普通任务也纳入 DL 带宽隔离

`update_curr()` 里有一行关键调用：

```c
dl_server_update(&rq->fair_server, delta_exec);
```

每个 CPU 的 `rq` 有一个 `fair_server`（是个 `sched_dl_entity`），**代表 fair 类去申请一块 DL 带宽**，再由它 `pick_task_fair()` 选 fair 任务。这样**没有带宽上限的 fair 类也被套上了可回收的 CBS 预算**——详见 [[linux/deadline|Linux DL 实现与注意]] §6。

---

## ⚠️ 12. 注意点（踩坑清单）

> [!WARNING] 高频坑
> 1. **排序键是 `deadline` 不是 `vruntime`**：读代码时别按经典 CFS 的直觉找"最左 vruntime"。
> 2. **`sched_latency` / `sched_min_granularity` 已不是核心参数**：EEVDF 用单一 `sched_base_slice`（0.7 ms）。
> 3. **`vlag` 是近似值且被 clamp**：不要把它当成精确的 $w_i(V-v_i)$；稳态界是 `-r_max < lag < max(r_max, q)`。
> 4. **`zero_vruntime` 有"左偏"约定**：`avg_vruntime()` 必须保证 `avg + 0` 判为合格，源码里 `runtime -= (weight - 1)` 就是为此。
> 5. **合格 ≠ 会被选中**：`pick_eevdf` 还要过 buddy 提示、保护片、与 `curr` 比 deadline。
> 6. **`SCHED_IDLE` 策略 ≠ IDLE 调度类**：前者是 fair 内的最弱权重（`WEIGHT_IDLEPRIO = 3`，`sched.h:2549`），后者是 `idle.c` 的独立类。见 [[linux/idle|Linux IDLE 实现]]。
> 7. **带宽节流会先尝试续额度**：`runtime_remaining` 扣光不等于立刻被节流，可能拿到新的 5ms 切片。
> 8. **延迟出队（`sched_delayed`）**：负 lag 的实体不立即出队，留在树上"烧掉"负 lag，避免不公平。

**调试入口**：

- `/proc/sched_debug` 的 `cfs_rq` 段：`nr_queued`、`h_nr_queued`、`curr->vruntime`、`curr->deadline`、`min_vruntime`；
- `sched_feat()` 各开关（`PLACE_LAG` / `RUN_TO_PARITY` / `PREEMPT_SHORT` / `DELAY_DEQUEUE` …）见 `kernel/sched/features.h`；
- `cpu.max` / `cpu.stat` 观测带宽节流次数与时长。

---

## 🧭 一句话总结

> **现代 fair 类 = EEVDF：用 `avg_vruntime` 算 $V$、用 `entity_lag` 判合格、用 `deadline` 排序（增广红黑树剪枝）选出"最早合格虚拟截止期"的实体；片耗尽由 `update_deadline` 推进，睡眠用 lag 保留补偿，抢占靠保护片 `vprot` 与短片插队，组调度与 `fair_server` 再套一层带宽隔离。**

---

## 🔗 相关

- [[principle/cfs|CFS 完全公平调度原理]]：公平度量、vruntime、lag 与虚拟截止期的理论
- [[linux/deadline|Linux DL 实现与注意]]：`fair_server` 背后的 DL server
- [[linux/stop|Linux STOP 实现与注意]]、[[linux/idle|Linux IDLE 实现与注意]]
