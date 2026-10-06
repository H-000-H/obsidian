---
title: Linux IDLE 实现与注意
tags:
  - linux-kernel
  - scheduler
  - idle
  - cpuidle
  - tickless
  - nohz
status: Completed
created: 2026-10-06
---

# 💤 Linux IDLE 实现与注意

> [!INFO] 笔记概览
> - **源码基线**：`torvalds/linux` master，**Linux 7.3.0-rc6**，commit `22430ae5d90a`（2026-10-06）。文件：`kernel/sched/idle.c`、`kernel/sched/core.c`（`init_idle`）、`kernel/sched/sched.h`。
> - **定位**：把 [[principle/idle|IDLE 调度类原理]] 落到 Linux 真实实现——IDLE 类管的是"每 CPU 那个永远就绪的兜底任务"，它把 CPU 送进低功耗（cpuidle + tickless）并保证一有活立刻醒。**源码开头就声明：与 `SCHED_IDLE` 策略无关**。
> - **内容范围**：文件地图 · idle 任务诞生（`init_idle`）· idle 循环 `do_idle` 全流程 · cpuidle 路径 · tickless 细节 · `sched_class` 方法表 · 与 `SCHED_IDLE` 策略的区别 · 注意点
> - **推荐路径**：§1 地图 → §2 任务诞生 → §3 idle 循环 → §4 cpuidle 路径 → §5 tickless → §6 方法表 → §7 与 SCHED_IDLE 的区别 → §8 注意点

---

## 🗺️ 1. 文件地图与调度类位置

### 1.1 文件分工

| 文件 | 职责 |
|---|---|
| `kernel/sched/idle.c` | idle **调度类** + **idle 循环**（`do_idle`、`cpu_startup_entry`、cpuidle 调用） |
| `kernel/sched/core.c:8317` | `init_idle()`：创建/初始化每 CPU idle 任务 |
| `kernel/sched/sched.h` | `idle_policy()`、`task_has_idle_policy()`、`WEIGHT_IDLEPRIO` |

> [!WARNING] 开篇声明
> `idle.c` 第 6-7 行明确注释：**这些与 `SCHED_IDLE` 批处理任务无关**，后者在 `sched/fair.c` 里处理。这是最容易混淆的一点。

### 1.2 位置

层级顺序（`core.c:8987` 起自检）：

```
stop > dl > rt > fair > ext > idle
```

IDLE 排**最后**，所以只要有**任何**更高类的任务可运行，idle 任务就选不上。

### 1.3 `idle.c` 关键位置

| 行号 | 函数 | 职责 |
|---|---|---|
| 22 | `sched_idle_set_state` | 记录本 CPU 进入的 idle state |
| 29 | `cpu_idle_poll_ctrl` | 强制轮询开关（`nohlt`/`hlt` 启动参数） |
| 57 | `cpu_idle_poll` | 轮询式空闲（开中断自旋） |
| 113 | `default_idle_call` | 无 cpuidle 框架时的默认入睡 |
| 143 | `call_cpuidle` | 调用 cpuidle 进指定状态 |
| 164 | `idle_call_stop_or_retain_tick` | 决定停/留 tick |
| 181 | `cpuidle_idle_call` | **cpuidle 主流程** |
| 278 | `do_idle` | **idle 循环主体** |
| 411 | `play_idle_precise` | 强制进入空闲一段精确时间（测试/省电） |
| 448 | `cpu_startup_entry` | 每个 CPU 的入口：`while(1) do_idle()` |
| 461-598 | 各类 `*_idle` | `sched_class` 方法 |

---

## 🧬 2. idle 任务的诞生：`init_idle`（`core.c:8317`）

内核为**每个 CPU** 静态创建一个 idle 任务（`fork_idle` → `init_idle`）：

```c
void __init init_idle(struct task_struct *idle, int cpu)
{
	...
	idle->__state = TASK_RUNNING;              /* 永远 RUNNING */
	idle->se.exec_start = sched_clock();
	idle->flags |= PF_KTHREAD | PF_NO_SETAFFINITY;
	kthread_set_per_cpu(idle, cpu);            /* 绑定到该 CPU */

	set_cpus_allowed_common(idle, &ac);        /* 亲和性 = 仅本 CPU */
	__set_task_cpu(idle, cpu);

	rq->idle = idle;
	rq_set_donor(rq, idle);
	rcu_assign_pointer(rq->curr, idle);
	idle->on_rq = TASK_ON_RQ_QUEUED;           /* 永远在队列上 */
	idle->on_cpu = 1;
	...
	/* The idle tasks have their own, simple scheduling class: */
	idle->sched_class = &idle_sched_class;

	vtime_init_idle(idle, cpu);
	sprintf(idle->comm, "%s/%d", INIT_TASK_COMM, cpu);   /* swapper/N */
}
```

要点：

- **`__state = TASK_RUNNING` + `on_rq = QUEUED`**：idle 任务**永远就绪**，是调度器的兜底——保证"只要 CPU 在线就一定选得出任务"；
- **`PF_NO_SETAFFINITY` + 仅本 CPU 亲和**：idle 任务**不可迁移**；
- **`sched_class = idle_sched_class`**：换上 IDLE 类；
- **comm = `swapper/N`**（PID 0，`INIT_TASK_COMM` 为 `swapper`）。

> [!NOTE] 为什么"永远就绪"是硬需求
> 调度器的基本假设是"总有东西可跑"。idle 任务充当这个"永远存在的下一个"。于是"省电"这件事被**封装进 idle 任务自己的代码**里，调度逻辑保持统一。

---

## 🔁 3. idle 循环：`do_idle`（`idle.c:278`）

### 3.1 入口

```c
void cpu_startup_entry(enum cpuhp_state state)
{
	current->flags |= PF_IDLE;
	arch_cpu_idle_prepare();
	cpuhp_online_idle(state);
	while (1)
		do_idle();
}
```

每个 CPU 起来后进入 `while(1) do_idle()`——**idle 任务永远在这个循环里**。

### 3.2 `do_idle` 全流程

```c
static void do_idle(void)
{
	int cpu = smp_processor_id();
	bool got_tick = false;

	if (cpu_is_offline(cpu)) {                  /* ① CPU 下线路径 */
		local_irq_disable();
		WARN_ON_ONCE(need_resched());
		cpuhp_report_idle_dead();
		arch_cpu_idle_dead();                    /* 不返回 */
	}

	nohz_run_idle_balance(cpu);                  /* ② 空闲负载均衡（tickless 下） */

	__current_set_polling();
	tick_nohz_idle_enter();                      /* ③ 进入 tickless idle */

	while (!need_resched()) {                    /* ④ 主循环 */
		local_irq_disable();                     /* 关中断，避免检查与入睡之间丢唤醒 */

		arch_cpu_idle_enter();
		rcu_nocb_flush_deferred_wakeup();

		if (cpu_idle_force_poll || tick_check_broadcast_expired()) {
			tick_nohz_idle_restart_tick();
			cpu_idle_poll();                     /* 轮询式：开中断自旋 */
		} else {
			cpuidle_idle_call(got_tick);         /* 真正入睡 */
		}
		got_tick = tick_nohz_idle_got_tick();
		arch_cpu_idle_exit();
	}

	/* ⑤ 退出空闲，回到调度 */
	preempt_set_need_resched();
	tick_nohz_idle_exit();
	__current_clr_polling();
	smp_mb__after_atomic();
	flush_smp_call_function_queue();
	schedule_idle();                             /* 真正切回调度器 */

	if (unlikely(klp_patch_pending(current)))
		klp_update_patch_state(current);
}
```

| 阶段 | 代码 | 作用 |
|---|---|---|
| ① | `cpu_is_offline` → `arch_cpu_idle_dead()` | **CPU 热插拔下线收尾**（不返回） |
| ② | `nohz_run_idle_balance` | 进低功耗前先"再抢救一下"（空闲负载均衡） |
| ③ | `tick_nohz_idle_enter` | 停掉周期 tick 的前置 |
| ④ | `while (!need_resched())` | 主循环：轮询 or 入睡，被中断唤醒后重查 |
| ⑤ | `tick_nohz_idle_exit` / `schedule_idle()` | 恢复 tick、切回调度 |

### 3.3 主循环里最微妙的地方

源码用一大段注释解释**"检查 → 入睡"之间不能有缝**：

> 从关中断那一刻起，直到 CPU 睡眠指令被执行，中断不应被重新打开。否则一个中断可能触发、排入一个定时器，而这个定时器直到 CPU 从睡眠指令醒来才被处理；而 `need_resched()` 检查**并不能反映有待处理的定时器重编程**。

于是区分了三种 arch 实现：

| 方案 | 是否安全 | 原因 |
|---|---|---|
| `wfi` / `mwait`（SLEEP-UNTIL-PENDING-INTERRUPT） | ✅ | 可在**关中断**下进入睡眠指令 |
| `sti; mwait` 组合 | ✅ | 只在执行 `mwait` 时开中断，无缝隙 |
| 回滚式 idle handler（睡眠指令**开中断**执行） | ❌ | 会漏掉定时器重编程；须改用 FAST-FORWARD |

> [!WARNING] "睡死"是设计大忌
> 若"检查完没人要我"与"真正睡下"之间来了个任务却丢了唤醒标志，CPU 就会睡过头。这就是为什么 `do_idle` 在检查前 `local_irq_disable()`。

### 3.4 轮询 vs 入睡

```c
if (cpu_idle_force_poll || tick_check_broadcast_expired()) {
	tick_nohz_idle_restart_tick();
	cpu_idle_poll();
} else {
	cpuidle_idle_call(got_tick);
}
```

- **轮询**（`cpu_idle_poll`，`idle.c:57`）：开中断空转 `cpu_relax()`，不省电，但省下"进出低功耗"的开销；
- 触发条件：`cpu_idle_force_poll`（`nohlt` 启动参数 / `cpu_idle_poll_ctrl`）或**广播定时器已过期**（知道马上有 IPI 来，不值得深睡）；
- **否则**走 cpuidle 真正入睡。

> [!TIP] 轮询有时反而更优
> 进/出深度睡眠状态本身有开销。马上就有活时，空转几微秒比"睡下再被叫醒"更划算——所以内核保留了强制轮询开关。

---

## ⚡ 4. cpuidle 路径：`cpuidle_idle_call`（`idle.c:181`）

```c
static void cpuidle_idle_call(bool stop_tick)
{
	struct cpuidle_device *dev = cpuidle_get_device();
	struct cpuidle_driver *drv = cpuidle_get_cpu_driver(dev);
	int next_state, entered_state;

	if (need_resched()) {                 /* 有活了，直接退出 */
		local_irq_enable();
		return;
	}

	if (cpuidle_not_available(drv, dev)) { /* 没有 cpuidle 框架 */
		idle_call_stop_or_retain_tick(stop_tick);
		default_idle_call();               /* 退化：arch_cpu_idle() */
		goto exit_idle;
	}

	if (idle_should_enter_s2idle() || dev->forced_idle_latency_limit_ns) {
		/* Suspend-to-idle：绕过 governor，直接选最深的 idle state */
		...
		tick_nohz_idle_stop_tick();
		next_state = cpuidle_find_deepest_state(drv, dev, max_latency_ns);
		call_cpuidle(drv, dev, next_state);
	} else if (drv->state_count > 1) {
		stop_tick = true;
		next_state = cpuidle_select(drv, dev, &stop_tick);   /* governor 决策 */
		idle_call_stop_or_retain_tick(stop_tick);
		entered_state = call_cpuidle(drv, dev, next_state);
		cpuidle_reflect(dev, entered_state);                 /* 反馈给 governor */
	} else {
		idle_call_stop_or_retain_tick(stop_tick);
		call_cpuidle(drv, dev, 0);                           /* 只有一个状态 */
	}

exit_idle:
	__current_set_polling();
	if (WARN_ON_ONCE(irqs_disabled()))
		local_irq_enable();
}
```

三条分支：

1. **无 cpuidle 框架** → `default_idle_call()`（`idle.c:113`）调 `arch_cpu_idle()`（如 ARM 的 `wfi`）；
2. **s2idle / 强制延迟上限** → 绕过 governor，直接选**能容忍的最深状态**；
3. **正常多状态** → `cpuidle_select()` 让 **governor 预测空闲时长**并选一个 C-state，进去后再 `cpuidle_reflect()` 反馈实际停留时间，供下次决策。

> [!NOTE] governor 的权衡
> 睡得越深越省电，但**退出延迟越大**。governor 的目标是：在"不拖慢唤醒"的前提下尽量省电。所以它要预测空闲时长，而不是无脑选最深的。

`call_cpuidle()`（`idle.c:143`）进入前会再确认一次 `current_clr_polling_and_test()`——若期间已有活，直接返回 `-EBUSY`，不进睡眠。

---

## ⏲️ 5. tickless 细节

| 调用 | 位置 | 作用 |
|---|---|---|
| `tick_nohz_idle_enter()` | `do_idle` ③ | 进入 tickless 状态 |
| `idle_call_stop_or_retain_tick(stop_tick)` | 164 | 按需停 tick 或保留 |
| `tick_nohz_idle_stop_tick()` | 164/228 | **停掉周期 tick** |
| `tick_nohz_idle_restart_tick()` | 352 | 轮询前恢复 tick |
| `tick_nohz_idle_exit()` | `do_idle` ⑤ | 退出 tickless |
| `tick_nohz_idle_got_tick()` | 357 | 本轮是否收到过 tick |

```c
static void idle_call_stop_or_retain_tick(bool stop_tick)
{
	if (stop_tick || tick_nohz_tick_stopped())
		tick_nohz_idle_stop_tick();
	else
		tick_nohz_idle_retain_tick();
}
```

- 传统内核每 CPU 都有**周期性 tick**，无事也唤醒——费电；
- **NOHZ** 空闲时停掉周期 tick，改为**按需**设置下一个定时器；于是"空闲"才真正空闲；
- `got_tick` 会在下一轮传给 `cpuidle_idle_call`——若上一轮已经收到过 tick，本轮倾向保留 tick（可能马上要跑）。

---

## 🧩 6. `sched_class` 方法表

```c
DEFINE_SCHED_CLASS(idle) = {
	/* no enqueue/yield_task for idle tasks */
	/* dequeue is not valid, we print a debug message there: */
	.dequeue_task		= dequeue_task_idle,
	.wakeup_preempt		= wakeup_preempt_idle,
	.pick_task		= pick_task_idle,
	.put_prev_task		= put_prev_task_idle,
	.set_next_task          = set_next_task_idle,
	.balance		= balance_idle,
	.select_task_rq		= select_task_rq_idle,
	.set_cpus_allowed	= set_cpus_allowed_common,
	.task_tick		= task_tick_idle,
	.prio_changed		= prio_changed_idle,
	.switching_to		= switching_to_idle,
	.update_curr		= update_curr_idle,
};
```

| 方法 | 实现 | 含义 |
|---|---|---|
| `pick_task_idle` | `return rq->idle;` | 被选中就返回本 CPU 的 idle 任务 |
| `wakeup_preempt_idle` | `resched_curr(rq);` | **无条件让位**（有活就重新调度） |
| `dequeue_task_idle` | 打印 `"bad: scheduling from the idle thread!"` + `dump_stack()` | **idle 里睡眠是非法** |
| `balance_idle` | `WARN_ON_ONCE(1)` | idle 任务**不参与负载均衡** |
| `select_task_rq_idle` | `return task_cpu(p);` | **不可迁移** |
| `switching_to_idle` | `BUG()` | 不可能"切进"这个类 |
| `prio_changed_idle` | 变了就 `BUG()` | 优先级不可改 |
| `task_tick_idle` | `update_curr_idle(rq)` | tick 时记账 |
| `update_curr_idle` | 记到 `dl_server` | 见下 |

### 6.1 "不能在 idle 里睡眠"的强制

```c
static bool
dequeue_task_idle(struct rq *rq, struct task_struct *p, int flags)
{
	raw_spin_rq_unlock_irq(rq);
	printk(KERN_ERR "bad: scheduling from the idle thread!\n");
	dump_stack();
	raw_spin_rq_lock_irq(rq);
	return true;
}
```

idle 任务一旦被"出队"（意味着有人试图在 idle 上下文里 `schedule()`），内核立即打印错误 + 栈回溯——这是原理篇 §6 "空闲任务里不能睡眠"的代码级强制。

### 6.2 `update_curr_idle`：idle 时间也要记账

```c
static void update_curr_idle(struct rq *rq)
{
	struct sched_entity *se = &rq->idle->se;
	u64 now = rq_clock_task(rq);
	s64 delta_exec;

	delta_exec = now - se->exec_start;
	if (unlikely(delta_exec <= 0))
		return;

	se->exec_start = now;

	dl_server_update_idle(&rq->fair_server, delta_exec);   /* 记到 fair_server */
#ifdef CONFIG_SCHED_CLASS_EXT
	dl_server_update_idle(&rq->ext_server, delta_exec);
#endif
}
```

> [!NOTE] 为什么 idle 时间要记到 DL server
> `fair_server` / `ext_server` 是 DL 带宽服务器（见 [[linux/deadline|Linux DL 实现与注意]] §6）。CPU 空闲时，fair/ext 类"没用掉的带宽"要被正确记账，才能做 GRUB 回收——所以 idle 的时长要**通知**这些 server。

---

## ⚠️ 7. 关键区别：IDLE 调度类 ≠ `SCHED_IDLE` 策略

| | **IDLE 调度类**（本篇） | **`SCHED_IDLE` 策略** |
|---|---|---|
| 归属文件 | `idle.c`（独立类） | `fair.c`（EEVDF 内部） |
| 优先级 | **最低的类**，低于 fair | **fair 类内最弱**，低于 `SCHED_NORMAL` |
| 载体 | 每 CPU 内核 idle 任务（PID 0） | 普通用户进程 |
| 权重 | 不走权重 | `WEIGHT_IDLEPRIO = 3`（`sched.h:2549`） |
| 判定 | `is_idle_task()` | `task_has_idle_policy()` → `idle_policy(p->policy)` |
| 作用 | 无事可做时进低功耗 | 只在"真没人要 CPU"时才跑的后台任务 |
| 用户可见 | 否 | 是（`chrt` / `sched_setscheduler`） |

```c
/* kernel/sched/sched.h:189 */
static inline int idle_policy(int policy)
{
	return policy == SCHED_IDLE;
}
```

> [!WARNING] 别把两者当一回事
> `SCHED_IDLE` 任务仍走 EEVDF（权重极低），**不会**让 CPU 进低功耗；让 CPU 进低功耗的是 **IDLE 调度类**的 idle 任务。

---

## ⚠️ 8. 注意点（踩坑清单）

> [!WARNING] 高频坑
> 1. **idle 里不能睡眠**：`dequeue_task_idle` 会打印 `bad: scheduling from the idle thread!` 并 `dump_stack()`。
> 2. **idle 任务不可迁移、不可改优先级**：`select_task_rq_idle` 返回原 CPU；`prio_changed_idle` / `switching_to_idle` 是 `BUG()`。
> 3. **idle 任务不参与负载均衡**：`balance_idle` 是 `WARN_ON_ONCE(1)`——均衡由"进 idle 前的 `nohz_run_idle_balance`"负责。
> 4. **`while(!need_resched())` 前必须关中断**：否则"检查—入睡"之间的唤醒会丢，导致睡死。
> 5. **轮询不等于省电**：`cpu_idle_force_poll` 会**放弃**省电，用于调试或"马上会醒"。
> 6. **CPU 下线复用 idle 路径**：`do_idle` 开头 `cpu_is_offline` 分支走 `arch_cpu_idle_dead()`。
> 7. **`play_idle_precise` 有前置条件**：要求 `SCHED_FIFO`、单 CPU 亲和、`PF_KTHREAD`、无 `mm` 等，否则 `WARN`。
> 8. **idle 时间要记到 DL server**：`update_curr_idle` 里 `dl_server_update_idle` 是 GRUB 回收的前提。

**调试入口**：

- `/proc/sched_debug` 中 idle 任务（`swapper/N`）的 `se.exec_start`；
- cpuidle：`/sys/devices/system/cpu/cpu*/cpuidle/` 下的 `state*/name`、`time`、`usage`；
- `nohlt` / `hlt` 启动参数控制轮询；`play_idle_precise()` 强制精确空闲（测试用）。

---

## 🧭 一句话总结

> **Linux IDLE = 每 CPU 一个 `init_idle` 造出来的 `swapper/N`（永远 RUNNING、永远在队列、不可迁移）+ `do_idle` 循环：先 `nohz_run_idle_balance` 抢救任务，再在 `while(!need_resched())` 里经 `cpuidle_idle_call` 入睡（或 `cpu_idle_poll` 空转），`tick_nohz_*` 停/恢复 tick；被唤醒后 `schedule_idle()` 让位。它管省电与唤醒，不管"调度谁跑"。**

---

## 🔗 相关

- [[principle/idle|IDLE 调度类原理]]：空闲任务、cpuidle、tickless 的理论
- [[linux/cfs|Linux CFS/EEVDF 实现与注意]]：`SCHED_IDLE` 策略与 `WEIGHT_IDLEPRIO`
- [[linux/stop|Linux STOP 实现与注意]]、[[linux/deadline|Linux DL 实现与注意]]
