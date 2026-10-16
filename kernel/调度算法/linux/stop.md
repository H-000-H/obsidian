---
title: Linux STOP 实现与注意
tags:
  - linux-kernel
  - scheduler
  - stop-task
  - stop-machine
  - cpu-hotplug
status: Completed
---

# 🛑 Linux STOP 实现与注意

> [!INFO] 笔记概览
> - **源码基线**：`torvalds/linux` master，**Linux 7.3.0-rc6**，commit `22430ae5d90a`（2026-10-06）。文件：`kernel/sched/stop_task.c`、`kernel/stop_machine.c`、`kernel/sched/core.c`、`kernel/sched/sched.h`。
> - **定位**：把 [[principle/stop|STOP 调度类原理]] 落到 Linux 真实实现——STOP 类本身**极简**（`stop_task.c` 仅 119 行，大量 `BUG()` 表示"不该发生"），真正的停机逻辑在 **`stop_machine.c`**：用每 CPU 一个 `migration/N` 内核线程 + 多核状态机实现"冻结一组 CPU"。
> - **内容范围**：文件地图 · STOP 任务从哪来（`smpboot` + `sched_set_stop_task`）· `sched_class` 方法表逐项解读 · 单核停机 · 多核停机状态机 · 各类 `stop_*` API · 实现约束 · 注意点
> - **推荐路径**：§1 地图 → §2 STOP 任务来源 → §3 方法表 → §4 单核停机 → §5 多核停机 → §6 API 家族 → §7 约束 → §8 注意点

---

## 🗺️ 1. 文件地图与调度类位置

### 1.1 两个文件分工

| 文件 | 职责 |
|---|---|
| `kernel/sched/stop_task.c` | STOP **调度类**本身：119 行，只描述"停机任务怎么被选中/切换" |
| `kernel/stop_machine.c` | 停机**机制**：每 CPU 停机线程、工作队列、单核/多核停机、状态机 |

> [!NOTE] 类与机制分离
> `stop_task.c` 是调度器的"插槽"，`stop_machine.c` 是使用方。停机任务本体就是 `stop_machine.c` 里创建的 `migration/N` 线程——它被 `sched_set_stop_task()` 换上了 `stop_sched_class`。

### 1.2 位置

层级顺序（`core.c:8987` 起 `BUG_ON` 自检）：

```
stop > dl > rt > fair > ext > idle
```

```c
BUG_ON(!sched_class_above(&stop_sched_class, &dl_sched_class));
BUG_ON(!sched_class_above(&dl_sched_class, &rt_sched_class));
BUG_ON(!sched_class_above(&rt_sched_class, &fair_sched_class));
BUG_ON(!sched_class_above(&fair_sched_class, &idle_sched_class));
```

STOP 排**最前**，所以只要 `rq->stop` 就绪，任何 CPU 必选它——"独占"靠的是**类优先级天然最高**，不是关抢占。

### 1.3 `stop_task.c` 关键位置

| 行号 | 符号 | 说明 |
|---|---|---|
| 13 | `select_task_rq_stop` | 永不迁移 |
| 19 | `balance_stop` | 只报告"是否 runnable" |
| 25 | `wakeup_preempt_stop` | **空函数**：从不被抢占 |
| 35 | `pick_task_stop` | 选 `rq->stop` |
| 44/50 | `enqueue_task_stop` / `dequeue_task_stop` | 仅加减 `nr_running` |
| 56 | `yield_task_stop` | `BUG()` |
| 74 | `task_tick_stop` | 空函数 |
| 78 | `switching_to_stop` | `BUG()`：不可能切进来 |
| 84 | `prio_changed_stop` | 改优先级即 `BUG()` |
| 99 | `DEFINE_SCHED_CLASS(stop)` | 方法表 |

---

## 🧬 2. STOP 任务从哪来

### 2.1 每 CPU 一个 `migration/N` 线程

`stop_machine.c` 用 `smpboot` 注册**每 CPU 热插拔线程**：

```c
static struct smp_hotplug_thread cpu_stop_threads = {
	.store			= &cpu_stopper.thread,
	.thread_should_run	= cpu_stop_should_run,
	.thread_fn		= cpu_stopper_thread,
	.thread_comm		= "migration/%u",     /* 线程名 */
	.create			= cpu_stop_create,
	.park			= cpu_stop_park,
	.selfparking		= true,
};

static int __init cpu_stop_init(void)
{
	for_each_possible_cpu(cpu) {
		struct cpu_stopper *stopper = &per_cpu(cpu_stopper, cpu);
		raw_spin_lock_init(&stopper->lock);
		INIT_LIST_HEAD(&stopper->works);
	}
	BUG_ON(smpboot_register_percpu_thread(&cpu_stop_threads));
	stop_machine_unpark(raw_smp_processor_id());
	stop_machine_initialized = true;
	return 0;
}
early_initcall(cpu_stop_init);
```

每个 CPU 一个 `struct cpu_stopper`：

```c
struct cpu_stopper {
	struct task_struct	*thread;      /* 就是那个 migration/N */
	raw_spinlock_t		lock;
	bool			enabled;
	struct list_head	works;        /* 待执行的停机工作项 */
	struct cpu_stop_work	stop_work;    /* for stop_cpus */
	unsigned long		caller;
	cpu_stop_fn_t		fn;
};
```

### 2.2 上线时换上 STOP 调度类

线程创建回调 `cpu_stop_create()` 调用：

```c
static void cpu_stop_create(unsigned int cpu)
{
	sched_set_stop_task(cpu, per_cpu(cpu_stopper.thread, cpu));
}
```

`core.c:3658` 的 `sched_set_stop_task()` 做三件事：

```c
struct sched_param param = { .sched_priority = MAX_RT_PRIO - 1 };
...
/* 1) 先伪装成 SCHED_FIFO，用户态工具看到的是熟悉的东西 */
sched_setscheduler_nocheck(stop, SCHED_FIFO, &param);

/* 2) 再换上真正的 stop 调度类 */
stop->sched_class = &stop_sched_class;

/* 3) 给它独立的 pi_lock 锁类，避免与 PI 链递归冲突 */
lockdep_set_class(&stop->pi_lock, &stop_pi_lock);
...
cpu_rq(cpu)->stop = stop;
```

> [!TIP] 为什么先伪装成 `SCHED_FIFO`
> 源码注释：让它在用户态工具（`ps`/`chrt`）眼里像个"已知的东西"，不至于让人困惑；PI 也能"大致工作"。但停机工作**不应依赖 PI**。

> [!NOTE] 停机线程平时在睡觉
> `migration/N` 平时 `kthread_park` 着（`stop_machine_park`），只有投了工作项才被唤醒。所以 `sched_stop_runnable()` 用 `task_on_rq_queued()` 判断：

```c
static inline bool sched_stop_runnable(struct rq *rq)
{
	return rq->stop && task_on_rq_queued(rq->stop);
}
```

---

## 🧩 3. `sched_class` 方法表逐项解读

```c
DEFINE_SCHED_CLASS(stop) = {
	.enqueue_task		= enqueue_task_stop,
	.dequeue_task		= dequeue_task_stop,
	.yield_task		= yield_task_stop,
	.wakeup_preempt		= wakeup_preempt_stop,
	.pick_task		= pick_task_stop,
	.put_prev_task		= put_prev_task_stop,
	.set_next_task          = set_next_task_stop,
	.balance		= balance_stop,
	.select_task_rq		= select_task_rq_stop,
	.set_cpus_allowed	= set_cpus_allowed_common,
	.task_tick		= task_tick_stop,
	.prio_changed		= prio_changed_stop,
	.switching_to		= switching_to_stop,
	.update_curr		= update_curr_stop,
};
```

逐项含义（对照原理篇 §2 的独占语义）：

| 方法 | 实现 | 对应语义 |
|---|---|---|
| `wakeup_preempt_stop` | `/* we're never preempted */` 空 | **不可被抢占** |
| `yield_task_stop` | `BUG()` | **不可 yield** |
| `select_task_rq_stop` | `return task_cpu(p)` | **不可迁移** |
| `switching_to_stop` | `BUG()` | 不可能"切进"这个类 |
| `prio_changed_stop` | 变了就 `BUG()` | 优先级不可改 |
| `balance_stop` | `return sched_stop_runnable(rq)` | 不参与负载均衡，只报状态 |
| `task_tick_stop` | 空 | 不需要 tick 抢占 |
| `update_curr_stop` | 空 | 不记 vruntime 账 |
| `pick_task_stop` | `if (!sched_stop_runnable(rq)) return NULL; return rq->stop;` | 就绪必选 |
| `enqueue/dequeue_task_stop` | `add/sub_nr_running(rq, 1)` | 只维护计数 |
| `set_next_task_stop` | `stop->se.exec_start = rq_clock_task(rq)` | 仅记录起跑时刻 |
| `put_prev_task_stop` | `update_curr_common(rq)` | 复用公共记账 |

> [!WARNING] 满屏 `BUG()` 是设计声明
> `yield_task_stop` / `switching_to_stop` / `prio_changed_stop` 的 `BUG()` 不是"没写完"，而是**断言这些路径永不该发生**。一旦触发，说明调度器核心逻辑出了严重问题。

---

## 🎯 4. 单核停机：`stop_one_cpu`（`stop_machine.c:138`）

**目标**：让**某一个** CPU 停下来执行一个函数。

```c
int stop_one_cpu(unsigned int cpu, cpu_stop_fn_t fn, void *arg)
{
	struct cpu_stop_done done;
	struct cpu_stop_work work = { .fn = fn, .arg = arg, .done = &done, .caller = _RET_IP_ };

	cpu_stop_init_done(&done, 1);
	if (!cpu_stop_queue_work(cpu, &work))
		return -ENOENT;
	/*
	 * In case @cpu == smp_processor_id() we can avoid a sleep+wakeup
	 * cycle by doing a preemption:
	 */
	cond_resched();
	wait_for_completion(&done.completion);
	return done.ret;
}
```

流程：

1. 构造一个工作项（函数 + 参数 + 完成量），`caller = _RET_IP_` 记录调用点（调试用）；
2. 投递到目标 CPU 的 `stopper->works`；
3. `cond_resched()`——**若目标就是本 CPU，这次抢占就让本 CPU 的停机线程接管**，省去睡眠唤醒；
4. 否则 `wait_for_completion()` 等目标 CPU 干完。

工作项由停机线程 `cpu_stopper_thread()` 取出执行（见 §7）。

---

## 🌐 5. 多核停机：状态机

### 5.1 难点

一组 CPU 必须**同时**进入停机，否则先停的等后停的，而后停的可能正被先停的需要的锁卡住 → **死锁**。为此引入多核同步状态机。

### 5.2 状态与数据

```c
enum multi_stop_state {
	MULTI_STOP_NONE,       /* 起始占位 */
	MULTI_STOP_PREPARE,    /* 等所有线程被调度到 */
	MULTI_STOP_DISABLE_IRQ,/* 关中断 */
	MULTI_STOP_RUN,        /* 执行目标函数 */
	MULTI_STOP_EXIT,       /* 退出 */
};

struct multi_stop_data {
	cpu_stop_fn_t		fn;
	void			*data;
	unsigned int		num_threads;    /* 参与线程数 */
	const struct cpumask	*active_cpus;
	enum multi_stop_state	state;
	atomic_t		thread_ack;     /* "到齐"计数 */
};
```

### 5.3 同步原语：最后一个 ack 的推进状态

```c
static void set_state(struct multi_stop_data *msdata,
		      enum multi_stop_state newstate)
{
	atomic_set(&msdata->thread_ack, msdata->num_threads);   /* 重置计数 */
	smp_wmb();                                              /* 保证顺序 */
	WRITE_ONCE(msdata->state, newstate);
}

/* Last one to ack a state moves to the next state. */
static void ack_state(struct multi_stop_data *msdata)
{
	if (atomic_dec_and_test(&msdata->thread_ack))
		set_state(msdata, msdata->state + 1);
}
```

> [!NOTE] "最后一个人"语义
> 每个 CPU 到达一个状态就 `ack` 一次；**只有最后一个 ack 的**把状态推进到下一阶段。于是"所有人都在场"被严格保证。

### 5.4 每个 CPU 跑的主循环

```c
static int multi_cpu_stop(void *data)
{
	struct multi_stop_data *msdata = data;
	enum multi_stop_state newstate, curstate = MULTI_STOP_NONE;
	int cpu = smp_processor_id(), err = 0;
	...
	local_save_flags(flags);

	/* Simple state machine */
	do {
		stop_machine_yield(cpumask);            /* 默认 cpu_relax() */
		newstate = READ_ONCE(msdata->state);
		if (newstate != curstate) {
			curstate = newstate;
			switch (curstate) {
			case MULTI_STOP_DISABLE_IRQ:
				local_irq_disable();
				hard_irq_disable();
				break;
			case MULTI_STOP_RUN:
				if (is_active)                /* 只有 active CPU 执行 fn */
					err = msdata->fn(msdata->data);
				break;
			default:
				break;
			}
			ack_state(msdata);                    /* 到齐就推进 */
		} else if (curstate > MULTI_STOP_PREPARE) {
			touch_nmi_watchdog();                 /* 防硬锁死误报 */
			rcu_momentary_eqs();                  /* 抑制 RCU stall 告警 */
		}
	} while (curstate != MULTI_STOP_EXIT);

	local_irq_restore(flags);
	return err;
}
```

要点：

- **PREPARE 阶段**：等所有线程都被调度到同一个循环里（这是防死锁的关键——先到的在原地自旋，不持锁）；
- **DISABLE_IRQ 阶段**：关中断，保证停机窗口不被打扰；
- **RUN 阶段**：只有 `is_active` 的 CPU 执行目标函数；
- 自旋时调 `touch_nmi_watchdog()` / `rcu_momentary_eqs()`，避免长自旋被误判为死机。

### 5.5 发起端

```c
int stop_machine_cpuslocked(cpu_stop_fn_t fn, void *data, const struct cpumask *cpus)
{
	struct multi_stop_data msdata = {
		.fn = fn, .data = data,
		.num_threads = num_online_cpus(),
		.active_cpus = cpus,
	};
	lockdep_assert_cpus_held();

	if (!stop_machine_initialized) {          /* 启动早期还没初始化 */
		WARN_ON_ONCE(msdata.num_threads != 1);
		local_irq_save(flags);
		hard_irq_disable();
		ret = (*fn)(data);                    /* 退化为"关中断直接跑" */
		local_irq_restore(flags);
		return ret;
	}

	set_state(&msdata, MULTI_STOP_PREPARE);   /* 起手 PREPARE */
	return stop_cpus(cpu_online_mask, multi_cpu_stop, &msdata);
}
```

> [!TIP] 启动早期的退化路径
> `cpu_stop_init` 是 `early_initcall`，之前若有人调 `stop_machine()`，只能"单核 + 关中断"直接跑（此时本就单核）。

---

## 🧰 6. API 家族一览

| API | 位置 | 作用 |
|---|---|---|
| `stop_one_cpu(cpu, fn, arg)` | 138 | 单核停机，同步等待 |
| `stop_one_cpu_nowait(cpu, fn, arg, work)` | 383 | 单核停机，异步（自带 work 结构） |
| `stop_two_cpus(cpu1, cpu2, fn, arg)` | 336 | **两核同时**停（`cpu_stop_queue_two_works`，按序取锁防死锁） |
| `stop_cpus(cpumask, fn, arg)` | 462 | 一组核停机（`stop_cpus_mutex` 串行化） |
| `stop_machine(fn, data, cpus)` | 621 | 通用多核停机，内部 `cpus_read_lock` |
| `stop_machine_cpuslocked(...)` | 585 | 调用方已持 `cpus_read_lock` 时用 |
| `stop_core_cpuslocked(cpu, ...)` | 639 | 停同核心的 SMT 兄弟（`smt_mask`） |
| `stop_machine_from_inactive_cpu(...)` | 681 | 从**未上线** CPU 发起（如 CPU 启动早期） |

> [!NOTE] 串行化
> `stop_cpus()` 用全局 `stop_cpus_mutex` 把并发停机调用**串行化**，避免多个停机操作互相踩。源码注释强调：这让 `fn` 里等待别的停机变得安全。

---

## 🔒 7. 停机线程的执行约束

`cpu_stopper_thread()`（`stop_machine.c:485`）是工作项的实际执行者：

```c
if (work) {
	cpu_stop_fn_t fn = work->fn;
	...
	/* cpu stop callbacks must not sleep, make in_atomic() == T */
	stopper->caller = work->caller;
	stopper->fn = fn;
	preempt_count_inc();          /* 人为抬高抢占计数 → 睡眠即报错 */
	ret = fn(arg);
	if (done) {
		if (ret)
			done->ret = ret;
		cpu_stop_signal_done(done);
	}
	preempt_count_dec();
	...
	WARN_ONCE(preempt_count(), "cpu_stop: %ps(%p) leaked preempt count\n", fn, arg);
}
```

关键设计：

- **`preempt_count_inc()`**：把停机回调伪装成"原子上下文"，任何睡眠/调度都会立刻报错——这正是原理篇 §6 "停机函数不可睡眠"的代码级强制；
- **`WARN_ONCE` 泄漏检测**：回调若没配对地改抢占计数，会打印 `leaked preempt count`；
- 回调必须**短小**、**不持可能被他人持有的锁**。

---

## ⚠️ 8. 注意点（踩坑清单）

> [!WARNING] 高频坑
> 1. **STOP 类无用户接口**：没有 `SCHED_STOP`；停机线程是内核的 `migration/N`。
> 2. **停机 ≠ 关中断**：关中断只挡中断；STOP 是在**调度层面**独占 CPU，粒度更大。
> 3. **回调里绝不能睡眠/拿锁**：`preempt_count_inc()` 会把它变成硬性约束，违反即报错。
> 4. **多核停机必须全员到场**：靠 PREPARE 自旋 + `thread_ack` 到齐推进；少一个就全体卡住。
> 5. **`num_threads = num_online_cpus()`**：即便只停一部分 CPU，也要**所有在线 CPU** 参与状态机（`active_cpus` 只决定谁执行 `fn`）。
> 6. **停机调用被全局串行化**：`stop_cpus_mutex`，高并发下可能排队。
> 7. **它是"最后手段"**：能用 RCU / IPI / per-CPU 变量解决的，别用停机（会带来延迟尖峰）。
> 8. **启动早期退化**：`stop_machine_initialized` 之前调用会走"单核 + 关中断"路径。

**调试入口**：

- `print_stop_info()`（`stop_machine.c:53`）会打印 `Stopper: <fn> <- <caller>`，定位卡在哪个停机函数；
- `ps` 里 `[migration/N]` 线程状态；停机时它会短暂 RUN。

---

## 🧭 一句话总结

> **Linux STOP = `stop_task.c` 里一个"永不被抢占、永不迁移、不可 yield"的极简调度类 + `stop_machine.c` 里每 CPU 的 `migration/N` 线程；单核靠 `stop_one_cpu` 投递工作项，多核靠 `multi_cpu_stop` 状态机（PREPARE→DISABLE_IRQ→RUN→EXIT，最后一个 ack 推进）让一组 CPU 同时冻结执行一段不可睡眠的代码。**

---

## 🔗 相关

- [[principle/stop|STOP 调度类原理]]：为什么需要停机、独占语义、多核停机难点
- [[linux/cfs|Linux CFS/EEVDF 实现与注意]]：fair 类实现
- [[linux/idle|Linux IDLE 实现与注意]]、[[linux/deadline|Linux DL 实现与注意]]
