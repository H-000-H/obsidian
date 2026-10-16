---
title: Linux sched_ext 实现与注意
tags:
  - linux-kernel
  - scheduler
  - sched-ext
  - bpf
  - dsq
  - scx
  - struct-ops
status: Completed
---

# 🧩 Linux sched_ext 实现与注意

> [!INFO] 笔记概览
> - **源码基线**：`torvalds/linux` master，**Linux 7.3.0-rc6**，commit `69f80fef3153`（2026-10-06）。文件：`kernel/sched/ext/ext.c`（9613 行）、`kernel/sched/ext/internal.h`（2278 行）、`kernel/sched/ext/{cid.c,sub.c,idle.c,arena.c,inlines.h,types.h,ext.h}`、`include/linux/sched/ext.h`、`kernel/sched/sched.h`、`Documentation/scheduler/sched-ext.rst`。
> - **定位**：把 [[principle/ext|sched_ext 可扩展调度原理]] 落到 Linux 真实实现——`scx_sched` 实例、`scx_rq` 每 CPU 状态、`scx_dispatch_q`（DSQ）、`ops_state` 归属状态机、`dispatch_one()` 调度循环、`finish_dispatch()` 分派提交、bypass 与 watchdog 兜底。
> - **内容范围**：文件地图 · 数据结构 · 生命周期（enable/disable）· 入队路径 · 选队/调度路径 · DSQ 分派提交 · slice/tick/抢占 · bypass · watchdog/错误回退 · 子调度器与 CID · 方法表逐项 · 注意点
> - **推荐路径**：§1 地图 → §2 数据结构 → §3 生命周期 → §4 入队 → §5 选队 → §6 DSQ 提交 → §7 slice/tick → §8 bypass → §9 回退 → §10 子调度器 → §11 方法表 → §12 注意点

---

## 🗺️ 1. 文件地图与调度类位置

### 1.1 各文件分工

| 文件 | 行数 | 职责 |
|---|---|---|
| `kernel/sched/ext/ext.c` | 9613 | **调度类本体**：DSQ、dispatch、enqueue、bypass、watchdog、enable/disable、kfunc、struct_ops |
| `kernel/sched/ext/internal.h` | 2278 | 内部结构：`sched_ext_ops`、`scx_sched`、`scx_sched_pcpu`、`ops_state`、各 `SCX_*` 枚举 |
| `include/linux/sched/ext.h` | 291 | `sched_ext_entity`（内嵌 `task_struct`）、`scx_dispatch_q`、DSQ ID 编码、`SCX_SLICE_*` |
| `kernel/sched/ext/cid.c` / `cid.h` | 1021 | CID（CPU id 抽象）拓扑与 `scx_cmask` 位图运算 |
| `kernel/sched/ext/sub.c` / `sub.h` | 2285 | **子调度器**：层级、能力（caps）委派、rescue |
| `kernel/sched/ext/idle.c` / `idle.h` | 1370 | 内置 idle 跟踪（每 CPU/每节点 idle cpumask） |
| `kernel/sched/ext/arena.c` / `arena.h` | 108 | BPF arena（cid-form 调度器用）映射 |
| `kernel/sched/ext/inlines.h` | 121 | 内联辅助（`scx_task_sched()`、`scx_bypassing()` 等） |
| `kernel/sched/ext/ext.h` | 85 | 对外声明：`scx_tick()`、`init_scx_entity()`、`scx_fork()`… |
| `kernel/sched/sched.h` | 3520 | `struct scx_rq`、`enum scx_rq_flags`、`scx_rq_rescue` |
| `Documentation/scheduler/sched-ext.rst` | — | 设计目标与用户接口文档 |

### 1.2 层级位置

```
stop > dl > rt > fair > ext > idle
```

`core.c` 启动时用 `BUG_ON(!sched_class_above(...))` 逐对自检。**ext 在 fair 之下、idle 之上**——这也是为什么"未加载 BPF 调度器时，`SCHED_EXT` 任务按 `SCHED_NORMAL` 交给 fair"。

### 1.3 开关与用户接口

需打开 `CONFIG_SCHED_CLASS_EXT=y`（外加 `CONFIG_BPF=y`、`CONFIG_BPF_SYSCALL=y`、`CONFIG_BPF_JIT=y`、`CONFIG_DEBUG_INFO_BTF=y`）。子调度器另有 `CONFIG_EXT_SUB_SCHED`、cgroup 支持有 `CONFIG_EXT_GROUP_SCHED`。

```bash
# make -j16 -C tools/sched_ext
# tools/sched_ext/build/bin/scx_simple
# cat /sys/kernel/sched_ext/state        # enabled
# cat /sys/kernel/sched_ext/root/ops     # simple
```

全局 sysfs 属性（`ext.c:5327`）：`state` / `switch_all` / `nr_rejected` / `hotplug_seq` / `enable_seq`；每个调度器实例有自己的 kobject（根为 `/sys/kernel/sched_ext/root/`），暴露 `ops`（名字）与 `events`（诊断计数器）。

---

## 🧩 2. 核心数据结构

### 2.1 `scx_sched`：一个 BPF 调度器实例（`internal.h:1556`）

这是整个子系统的中枢。同一份内存**用 union 同时表示 cpu-form 与 cid-form 两套 ops**：

```c
struct scx_sched {
    union {
        struct sched_ext_ops        ops;      /* cpu-form */
        struct sched_ext_ops_cid    ops_cid;  /* cid-form：set_cmask/enable/select_cid… */
    };
    bool                    is_cid_type;
    bool                    dead;           /* ops.exit() 之后置位，门控 scx_prog_sched() */

    struct bpf_map          *arena_map;     /* cid-form 必须用且仅用一个 arena */
    struct gen_pool         *arena_pool;
    uintptr_t               arena_kern_base;

    DECLARE_BITMAP(has_op, SCX_OPI_END);    /* 哪些回调被实现 */

    struct rhashtable       dsq_hash;       /* user DSQ 按 id 哈希 */
    struct scx_sched_pnode  **pnode;        /* 每 NUMA 节点一个 global DSQ 分片 */
    struct scx_pshard       **pshard;       /* 每 shard 的能力位图（子调度器） */
    struct scx_sched_pcpu __percpu *pcpu;   /* 每 CPU 的调度器状态 */

    u64                     slice_dfl;
    u64                     bypass_timestamp;
    s32                     bypass_depth;   /* >0 表示处于 bypass */

    unsigned long           watchdog_timeout;
    atomic_t                exit_kind;
    struct scx_exit_info    *exit_info;

    struct kobject          kobj;           /* /sys/kernel/sched_ext/<name> */
    struct kthread_worker   *helper;        /* enable/disable 用 RT kthread */
    ...
    struct scx_sched        *ancestors[];   /* 含自身的祖先链 */
};
```

要点：

- **全局有多个 `scx_sched`**（根 + 子调度器），`scx_root` 是根实例（`ext.c:35`）；当前代码处于"多调度器支持"的过渡期，很多裸解引用仍是临时的 `scx_root`。
- `has_op` 位图让热路径用 `SCX_HAS_OP(sch, enqueue)` **一次判位**决定是否回调，避免间接调用开销。
- `bypass_depth` 支持**嵌套 bypass**（本身 bypass 或任一祖先 bypass 都算）。

### 2.2 `scx_rq`：每 CPU 的 sched_ext 状态（`sched.h:811`）

```c
struct scx_rq {
    struct scx_dispatch_q   local_dsq;        /* CPU 真正取任务的地方 */
#ifdef CONFIG_EXT_SUB_SCHED
    struct scx_dispatch_q   reject_dsq;       /* 能力被拒任务的暂存 */
    struct scx_rq_rescue    rescue;
#endif
    struct list_head        runnable_list;    /* 本 rq 上所有就绪任务 */
    unsigned long           ops_qseq;         /* 生成 ops_state 的 qseq */
    u32                     nr_running;
    u32                     cpuperf_target;
    bool                    in_select_cpu;
    bool                    cpu_released;     /* 是否已通知 BPF 释放核心 */
    u32                     flags;            /* SCX_RQ_* */
    u32                     nr_immed;         /* local_dsq 上的 IMMED 任务数 */
    u64                     clock;            /* 本 rq 时钟，scx_bpf_now() 用 */
    ...
    struct irq_work         deferred_irq_work;
    struct irq_work         kick_cpus_irq_work;
};
```

`enum scx_rq_flags`（`sched.h:782`）里几个关键位：`SCX_RQ_ONLINE`、`SCX_RQ_CAN_STOP_TICK`（可停 tick）、`SCX_RQ_IN_DISPATCH`、`SCX_RQ_IN_WAKEUP`、`SCX_RQ_BAL_CB_PENDING`。

> `scx_rq` 被**塞进 `struct rq`**（`rq->scx`），所以 sched_ext 的状态天然是每 CPU 的，无需额外查找。

### 2.3 `scx_dispatch_q`：DSQ（`include/linux/sched/ext.h:83`）

```c
struct scx_dispatch_q {
    raw_spinlock_t          lock;
    struct task_struct __rcu *first_task;  /* 无锁 peek 队首 */
    struct list_head        list;          /* 按分派顺序（FIFO 或 vtime 有序） */
    struct rb_root          priq;          /* 按 p->scx.dsq_vtime 排序 */
    u32                     nr;
    u32                     seq;           /* BPF 迭代器用 */
    u64                     id;
    struct rhash_head       hash_node;
    struct llist_node       free_node;
    struct scx_sched        *sched;
    struct scx_dsq_pcpu __percpu *pcpu;
    struct rcu_head         rcu;
};
```

**DSQ ID 是 64 位编码**（`include/linux/sched/ext.h:53`）：

```
Bits: [63] [62 .. 0]        [63]=1 内置，=0 BPF 自建 user DSQ
      [ B] [   ID   ]
内置 ID： [63][62 L][61..32 R][31..0 V]
  L=1 → LOCAL_ON（V 是 CPU 号）；内置值：
  SCX_DSQ_GLOBAL / SCX_DSQ_LOCAL / SCX_DSQ_BYPASS / SCX_DSQ_REJECT / SCX_DSQ_RESCUE
```

- 内置 DSQ **永远 FIFO**；`SCX_DSQ_LOCAL` 实际会被解析成"某 CPU 的 `rq->scx.local_dsq`"；
- `priq` 与 `list` **同时维护**：vtime 插入时先 `rb_add()` 再按前驱位置插链表，保证 `list` 也 vtime 有序（`ext.c:1640` 附近）。

### 2.4 `sched_ext_entity`：内嵌在 `task_struct`（`include/linux/sched/ext.h:185`）

`p->scx` 保存任务的 sched_ext 相关状态：

| 字段 | 作用 |
|---|---|
| `dsq` / `dsq_list` / `dsq_priq` | 当前所在的 DSQ 与链表/红黑树节点 |
| `ops_state` | **归属状态机**（NONE/QUEUEING/QUEUED/DISPATCHING + qseq） |
| `slice` | 运行时预算（ns），执行时递减 |
| `dsq_vtime` | PRIQ 排序键 |
| `flags` | `SCX_TASK_QUEUED`、`SCX_TASK_IN_CUSTODY`、`SCX_TASK_IMMED`、状态位（NONE→ENABLED→DEAD） |
| `runnable_at` | 变就绪的时刻（watchdog 用） |
| `tid` | fork 时分配的**唯一非零任务 ID**（`SCX_OPS_TID_TO_TASK`） |

**状态位**（`SCX_TASK_STATE_SHIFT` 起 3 位，`ext.h:118`）：

```
NONE → INIT_BEGIN → INIT → READY → ENABLED → DEAD
```

### 2.5 每 CPU / 每节点 / dispatch 缓冲

- `scx_sched_pcpu`（`internal.h:1348`）：每 CPU、每调度器状态。含 `cpus_to_kick` / `cpus_to_preempt` 等 kick 位图、`bypass_dsq`（bypass 专用队列）、`event_stats`，末尾是 **flex array** `dsp_ctx`。
- `scx_dsp_ctx`（`internal.h:1336`）：`ops.dispatch()` 里的**分派缓冲**，`buf[]` 暂存待提交任务（见 §6）。
- `scx_sched_pnode`（`internal.h:1412`）：每 NUMA 节点一个 `global_dsq` 分片，避免 bypass 时全局队列活锁。

### 2.6 `ops_state` 归属状态机（`internal.h:1990`）

```c
enum scx_ops_state {
    SCX_OPSS_NONE,        /* 核心拥有 */
    SCX_OPSS_QUEUEING,    /* 正交给 BPF（瞬态，持 rq 锁） */
    SCX_OPSS_QUEUED,      /* BPF 拥有 */
    SCX_OPSS_DISPATCHING, /* 正从 BPF 交还核心 */
    SCX_OPSS_QSEQ_SHIFT = 2,
};
```

`ops_state` 是 `atomic_long_t`：低 2 位是状态，**高位是 qseq**（32 位机上留 30 位）。qseq 用来在 **dispatch 与 dequeue/重新入队竞争**时判断"这次分派是否还认领得上任务"（`finish_dispatch()` 里比对，`ext.c:2862`）。

---

## 🔄 3. 生命周期：enable / disable

### 3.1 状态机（`internal.h:1869`）

```
SCX_DISABLED → SCX_ENABLING → SCX_ENABLED → SCX_DISABLING → SCX_DISABLED
```

对外可读为 `state` 的字符串：`disabled` / `enabling` / `enabled` / `disabling`。

### 3.2 注册（struct_ops）

BPF 侧用 **`bpf_struct_ops`** 注册：

- cpu-form：`bpf_sched_ext_ops`（`ext.c:8392`），名字 `sched_ext_ops`；
- cid-form：`bpf_sched_ext_ops_cid`（`ext.c:8465`），名字 `sched_ext_ops_cid`，供子调度器/多调度器使用。

两个 struct_ops 共享大部分 hook，`scx_init()`（`ext.c:11028`）里用 `BUILD_BUG_ON` 校验 union 两视图的字段偏移一致。

### 3.3 为什么 enable 要交给 RT kthread

`scx_enable()`（`ext.c:7975`）**被 offload 到专用系统级 RT kthread**（`scx_enable_cmd`，`internal.h:2037`）。原因（源码注释）：READY→ENABLED 切换任务时，**调用线程自己的 sched_class 会从 fair 变成 ext**，而 fair 优先级高于 ext——若 fair 类饱和，调用线程可能被**无限饿死**导致系统挂起。

### 3.4 切换范围

- 未设 `SCX_OPS_SWITCH_PARTIAL`：加载后 `SCHED_NORMAL/BATCH/IDLE/EXT` **全部**交给 sched_ext（`scx_switching_all`）；
- 设了：**只**接管 `SCHED_EXT`，其余仍归 fair；
- 停止（用户退出 / `SysRq-S` / 内部错误）→ `scx_disable_workfn()`（`ext.c:6706`）把**所有任务搬回 fair**。

> `enable_seq` 是"自启动以来是否加载过"的单调计数器（0 表示从未加载）。

---

## ⬇️ 4. 入队路径

### 4.1 `enqueue_task_scx()`（`ext.c:2149`）

```
enqueue_task_scx()
  ├─ 已在 QUEUED？→ 直接返回（幂等）
  ├─ 置 SCX_TASK_QUEUED，rq->scx.nr_running++，add_nr_running()
  ├─ ops.runnable()（若实现且非迁移中）
  ├─ 第一个任务 → dl_server_start(&rq->ext_server)   ← 见 §11
  └─ scx_do_enqueue_task(rq, p, enq_flags, sticky_cpu)
```

### 4.2 `scx_do_enqueue_task()`（`ext.c:1986`）——**分流决策**

这是理解 sched_ext 入队的关键，它决定任务**去 BPF 还是直接进本地队列**：

| 条件 | 去向 | 说明 |
|---|---|---|
| `sticky_cpu == cpu_of(rq)` | **local_norefill** | 内部搬迁/RESTORE，直接进本地 DSQ |
| `!scx_rq_online(rq)` | local | CPU 对 BPF 而言离线，别打扰 BPF |
| `scx_bypassing()` | **bypass** | 进 bypass DSQ（不回调 BPF） |
| `ddsp_dsq_id != SCX_DSQ_INVALID` | direct | 已直投 |
| `PF_EXITING` 且未设 `ENQ_EXITING` | local | 退出中任务自动兜底 |
| `is_migration_disabled()` 且未设 `ENQ_MIGRATION_DISABLED` | local | 不可迁移任务兜底 |
| 未实现 `enqueue` | **global** | 直接进 global DSQ |
| 否则 | **回调 BPF** | `SCX_CALL_OP_TASK(sch, enqueue, rq, p, enq_flags)` |

回调 BPF 的分支会做**归属交接**：

```c
qseq = rq->scx.ops_qseq++ << SCX_OPSS_QSEQ_SHIFT;
atomic_long_set(&p->scx.ops_state, SCX_OPSS_QUEUEING | qseq);
*ddsp_taskp = p;                       /* 允许 enqueue 内直投 */
SCX_CALL_OP_TASK(sch, enqueue, rq, p, enq_flags);
*ddsp_taskp = NULL;
if (p->scx.ddsp_dsq_id != SCX_DSQ_INVALID)
    goto direct;                       /* enqueue 里做了 direct dispatch */
p->scx.flags |= SCX_TASK_IN_CUSTODY;   /* 进入 BPF 保管 */
atomic_long_set_release(&p->scx.ops_state, SCX_OPSS_QUEUED | qseq);
```

要点：

- **`QUEUEING → QUEUED`** 的 `store_release` 与 dispatch/dequeue 的 `load_acquire` 配对，保证内存序；
- `SCX_TASK_IN_CUSTODY` 表示"任务在 BPF 手里"，只有离开保管时才回调 `ops.dequeue()`；
- 若 BPF 在 `enqueue()` 里直接 `scx_bpf_dsq_insert(p, SCX_DSQ_LOCAL, ...)`，走 **direct dispatch** 快路径（`direct_dispatch()`）。

### 4.3 `dequeue_task_scx()` / `ops_dequeue()`（`ext.c:2281` / `2204`）

出队先设 `SCX_DEQ_SCHED_CHANGE`（非睡眠出队），再 `ops_dequeue()`：

- 依据 `ops_state` 处理：`QUEUED` 时用 `cmpxchg` 抢到 `NONE`；`DISPATCHING` 时**等待**（`wait_ops_state`）——因为此刻别人正在把任务交给 DSQ，必须等它完成；
- 只有在任务**仍在 BPF 保管**（`task_leave_custody(p)`）时才回调 `ops.dequeue()`；
- 注释明确：`dequeue` 总是"临时的"，**不重置 `runnable_at`**。

---

## ⬆️ 5. 选队 / 调度路径

### 5.1 `pick_task_scx()` → `do_pick_task_scx()`（`ext.c:3440` / `3377`）

```c
static struct task_struct *pick_task_scx(struct rq *rq, struct rq_flags *rf)
{
    return do_pick_task_scx(rq, rf, false);
}
```

`do_pick_task_scx()` 流程：

```
do_pick_task_scx()
  ├─ dispatch_core_pick() 或 dispatch_pick()      ← 跑一次 dispatch
  ├─ verdict==RETRY → 返回 RETRY_TASK
  ├─ 若更高优先级类在 dispatch 期间入队（!force_scx）→ RETRY_TASK
  ├─ verdict==PREV → 继续跑 prev；切片没了就 refill 默认切片
  └─ 否则 → p = first_local_task(rq)；没有就返回 NULL
```

- **`force_scx=true`** 用于 `ext_server_pick_task()`（`ext.c:3451`）——DL server 必须选到 ext 任务；
- `first_local_task()`（`ext.c:3287`）取 `rq->scx.local_dsq` 队首——**CPU 只从这里取任务**；
- 若取到的任务**切片为 0**，内核用默认切片 refill 并打印一次 `has zero slice` 警告（`warned_zero_slice` 去重）。

### 5.2 `dispatch_one()`（`ext.c:2933`）——CPU 找活的完整流程

```
dispatch_one(rq, prev)
  ├─ 置 SCX_RQ_IN_DISPATCH
  ├─ prev 是 ext 且 runnable 且有切片且非 bypass → verdict=PREV（继续跑）
  ├─ rq->scx.local_dsq 非空 → verdict=LOCAL
  ├─ scx_dispatch_sched(root_sch, rq, prev, false)   ← 让 BPF 补货
  ├─ 没补到 → 若 prev 还 runnable 且允许保留 → verdict=PREV
  └─ 否则 → verdict=NONE
```

- `scx_dispatch_sched()` 会**递归**让当前调度器及其子调度器的 `ops.dispatch()` 跑（顶层到叶子一趟完成，见 §10）；
- `dispatch_pick()`（`ext.c:3296`）包住它并处理"**延迟的 balance callback**"：dispatch 可能要放 rq 锁，直接等会和给我们发 IPI（如 TLB flush）的 CPU 死锁，所以推迟到 balance callback 里等（`kick_sync_wait_bal_cb`）；
- `dispatch_core_pick()`（`ext.c:3328`，core scheduling 开启时）要求"选核原子"：一旦 dispatch 放锁就 `SCX_DSP_RETRY` 重来（比对 `lock_drop_seq`）。

### 5.3 `set_next_task_scx()`（`ext.c:3024`）

任务被选中后：若还在 `QUEUED`（core-sched 可能提前执行），补一次 `ops_dequeue(SCX_DEQ_CORE_SCHED_EXEC)` + `scx_dispatch_dequeue()`；然后回调 `ops.running()`、清 runnable、应用 out-of-band slice、刷新 `SCX_RQ_CAN_STOP_TICK`（`slice==SCX_SLICE_INF` 时可停 tick）。

---

## 📤 6. DSQ 分派提交：`scx_bpf_dsq_insert()` → `finish_dispatch()`

### 6.1 为什么要"两段式"

BPF 在 `ops.dispatch()` 里调 `scx_bpf_dsq_insert()`（kfunc，`ext.c:8927`）把任务插入 DSQ。**但 dispatch 到 local DSQ 可能需要等锁/换锁**——这些不能在 `ops.dispatch()` 里做（会锁序反转）。于是**分两段**（`ext.c:2826` 注释）：

1. `scx_bpf_dsq_insert()`：**只记录**任务与 qseq 到 `scx_dsp_ctx.buf[]`；
2. `ops.dispatch()` 返回后，`scx_flush_dispatch_buf()`（`ext.c:2904`）→ 逐条 `finish_dispatch()`（`ext.c:2837`）**真正提交**。

### 6.2 `finish_dispatch()` 的认领逻辑

```
finish_dispatch(sch, rq, p, qseq_at_dispatch, dsq_id, ...)
  ├─ opss==DISPATCHING/NONE → 别人已处理，返回
  ├─ opss==QUEUED 且 qseq 不匹配 → 任务已历经一轮，放弃
  ├─ cmpxchg QUEUED→DISPATCHING 抢认领；失败则 retry
  ├─ opss==QUEUEING → wait_ops_state() 后 retry
  └─ dsq = find_dsq_for_dispatch(...)
       ├─ LOCAL → dispatch_to_local_dsq()   （可能要换锁/迁移）
       └─ 其他  → scx_dispatch_enqueue()
```

- **`SCX_EV_INSERT_NOT_OWNED`**：若任务已不属于本调度器，计一次事件后放弃（BPF 允许"乱分派"，内核负责丢弃）；
- `find_dsq_for_dispatch()`（`ext.c:1831`）把 `SCX_DSQ_LOCAL` 解析成具体 CPU 的本地队列；`SCX_DSQ_REJECT/RESCUE` 是内部值。

### 6.3 `scx_dispatch_enqueue()`（`ext.c:1592`）

真正把任务挂到 DSQ：

- 内置 DSQ 上禁止 vtime 排序（会 `scx_error`）——避免 FIFO 任务饿死 PRIQ 任务；
- `SCX_ENQ_APPLY_SLICE` 时提交 slice/vtime；
- `SCX_ENQ_DSQ_PRIQ` → 红黑树 + vtime 有序链表；否则 FIFO 插头/尾（`SCX_ENQ_HEAD`/`SCX_ENQ_PREEMPT` 插头）；
- 头插时用 `rcu_assign_pointer(dsq->first_task, p)` 维护无锁 peek。

### 6.4 跨 CPU 直投与迁移

- `dispatch_to_local_dsq()`（`ext.c:2736`）：目标是**别的 CPU** 的本地 DSQ 时，需要"锁舞"——先记 `holding_cpu`、把 `ops_state` 清成 `NONE`、放当前 rq 锁、锁目标 rq、校验（亲和性/不可迁移）、必要时 `move_remote_task_to_local_dsq()`（`ext.c:2423`）完成 deactivate/activate 迁移；
- 任一步校验失败就**放弃迁移**，任务留在原地或落到 fallback DSQ。

---

## ⏱️ 7. slice / tick / 抢占

### 7.1 `update_curr_scx()`（`ext.c:1399`）

```c
delta_exec = update_curr_common(rq);
if (curr->scx.slice != SCX_SLICE_INF)
    curr->scx.slice -= min_t(u64, curr->scx.slice, delta_exec);
dl_server_update(&rq->ext_server, delta_exec);   /* 给 ext_server 记账 */
```

**与 fair/RT 不同**：ext 不记 vruntime、不做公平排序，`update_curr_scx()` 只做两件事——**递减切片**、**给 DL server 记账**。

### 7.2 `task_tick_scx()`（`ext.c:3826`）

```
update_curr_scx()
  ├─ bypassing → scx_set_task_slice(curr, 0)     （不可信 BPF，强制让出）
  ├─ 否则若实现 ops.tick() → 回调
  └─ 切片为 0 → resched_curr(rq)
```

### 7.3 抢占

- `wakeup_preempt_scx()`（`ext.c:2369`）：内核在唤醒点判断是否抢占；
- BPF 用 `SCX_ENQ_PREEMPT` 请求"插入即抢占"：把当前任务切片清零 + `resched_curr()`（`ext.c:1583` 附近）；
- `scx_bpf_kick_cpu()`（kfunc，内部实现 `scx_kick_cpu()`，`ext.c:9621`）：踢某 CPU 去重新调度（如刚给它本地队列放了任务）。

### 7.4 默认切片与 bypass 切片

| 常量 | 值 | 用途 |
|---|---|---|
| `SCX_SLICE_DFL` | 20 ms | BPF 忘了给切片时的 refill 默认值 |
| `SCX_SLICE_BYPASS` | 5 ms | bypass 模式的固定短切片 |
| `SCX_SLICE_INF` | `U64_MAX` | 无限切片（隐含 nohz，可停 tick） |

---

## 🚧 8. bypass（旁路）模式

`scx_bypass()`（`ext.c:6155`）：

1. 用 `scx_bypass_lock` 保护，`inc/dec_bypass_depth()`；
2. bypass 状态**沿调度器树向下传播**（本身或任一祖先 bypass 都算）；
3. 遍历**所有可能 CPU**（不能信任 BPF，故不用 `cpus_read_lock()`），逐个 rq：
   - 置/清 `SCX_SCHED_PCPU_BYPASSING`；
   - 沿 `rq->scx.runnable_list` **逆序**重排队任务（因为任务会被移出再插回，逆序遍历才能访问全部节点）；
   - bypass 时**清除保护**（`scx_task_slice_ended`）——"bypass 压过 protection"。

bypass 的语义：**不信任 BPF**，所有任务进 bypass DSQ 用固定短切片轮转，只求"**都能前进**"。

---

## 🐕 9. watchdog / 错误与回退

### 9.1 看门狗

- `check_rq_for_timeouts()`（`ext.c:3738`）：遍历 `rq->scx.runnable_list`，若某任务 `jiffies > runnable_at + watchdog_timeout` → `__scx_exit(SCX_EXIT_ERROR_STALL)`；
  - **归责**：若任务卡在"别人的 DSQ"上（如祖先的 bypass DSQ），**怪那个 drainer**（`dsq->sched`），本地 DSQ 则怪任务属主；
- `scx_watchdog_workfn()`（`ext.c:3782`）：一个**全局延迟工作**，定期扫所有在线 CPU；
- `scx_tick()`（`ext.c:3801`）：还有**二级看门狗**——若"看门狗自己"超过 `watchdog_timeout` 没 check-in，直接 `scx_exit(SCX_EXIT_ERROR_STALL)`。

### 9.2 退出类型（`internal.h:45`）

| 类别 | 取值 |
|---|---|
| 正常 | `SCX_EXIT_UNREG` / `UNREG_BPF` / `UNREG_KERN` / `SYSRQ` / `PARENT` / `PARENT_KILL` |
| 错误 | `SCX_EXIT_ERROR` / `ERROR_BPF` / `ERROR_STALL` / `ERROR_REENQ` / `ERROR_RESCUE` |

- **REENQ**：任务反复被重新入队却始终不运行（超 `SCX_REENQ_MAX_REPEAT`，`ext.c:2018`）→ BPF 在反复决定一个做不到的安置，踢掉；
- **RESCUE**：占用 rescue 执行超限被弹出（子调度器场景）。

### 9.3 回退的可靠性

退出时任务可能已**丢失 PID**（`PF_EXITING` 之后）。因此内核维护一份**独立的 `scx_tasks` 链表**（`ext.c:70`），记录**从 fork 到最终释放之间**的每个任务，保证 disable 时**任何状态都能遍历并搬回 fair**。

---

## 🧬 10. 子调度器与 CID（`CONFIG_EXT_SUB_SCHED`）

> 这是本基线的**进阶特性**：多个 BPF 调度器**分层共存**，按 cgroup 归属挂成树。

### 10.1 CID：CPU 的抽象编号

- `scx_cid_topo` / `scx_cmask`（`types.h`）：把 CPU 编号抽象成 **cid**，支持"每调度器可见的 CPU 子集"；
- `scx_cmask` 是位图（`bits[]`），`cid.h` 提供 `and/or/andnot/subset/intersects` 等运算；
- `scx_cpu_to_cid()` / `scx_cid_to_cpu()` 做双向映射。

### 10.2 能力（caps）委派

`enum scx_cap_flags`（`internal.h:1463`）：`ENQ_IMMED` / `ENQ` / `PREEMPT` / `PERF`。父调度器用 `scx_bpf_sub_grant()` / `scx_bpf_sub_revoke()` 把能力**委派给直接子节点**：

- 子节点的能力集**永远是父的子集**；
- 叶子在入队时选 cid，**取决于它被允许用哪些 cid**；
- `scx_pshard`（`internal.h:1531`）按 shard 分锁，不同 shard 不竞争。

### 10.3 dispatch 是"自顶向下递归"

与 enqueue 的"本地能力检查"不同，**dispatch 走 `scx_bpf_sub_dispatch()`**：父的 `dispatch` 回调在本地 rq 上**调用子节点的 dispatch**，整个子树**一趟分派完**（`internal.h:1430` 注释）。

### 10.4 rescue（救援）

`scx_rq_rescue`（`sched.h:800`）：当子调度器把任务"搁浅"（stranded）时，用**令牌桶**限制的救援执行把任务捞出来跑，超限则**弹出（eject）该子调度器**（`SCX_EXIT_ERROR_RESCUE`）。

---

## 📋 11. 方法表逐项（`DEFINE_SCHED_CLASS(ext)`，`ext.c:5076`）

| 方法 | 实现 | 作用 |
|---|---|---|
| `enqueue_task` | `enqueue_task_scx` | 入队（分流见 §4） |
| `dequeue_task` | `dequeue_task_scx` | 出队（临时出队不重置 runnable_at） |
| `yield_task` | `yield_task_scx` | `sched_yield()` |
| `yield_to_task` | `yield_to_task_scx` | 让给指定任务 |
| `wakeup_preempt` | `wakeup_preempt_scx` | 唤醒点抢占判断 |
| `pick_task` | `pick_task_scx` | **选下一个任务**（§5） |
| `put_prev_task` / `set_next_task` | `put_prev_task_scx` / `set_next_task_scx` | 换入换出通知 |
| `select_task_rq` | `select_task_rq_scx` | 选核入口（回调 `ops.select_cpu`） |
| `task_woken` | `task_woken_scx` | 唤醒后（配合 queued wakeup） |
| `set_cpus_allowed` | `set_cpus_allowed_scx` | 亲和性变更（回调 `set_cpumask`） |
| `rq_online` / `rq_offline` | `rq_online_scx` / `rq_offline_scx` | 置/清 `SCX_RQ_ONLINE`，offline 时 flush rescue |
| `task_tick` | `task_tick_scx` | tick（§7.2） |
| `switching_to` / `switched_from` / `switched_to` | `*_scx` | 调度类切换通知 |
| `reweight_task` | `reweight_task_scx` | 权重变更（回调 `set_weight`） |
| `prio_changed` | `prio_changed_scx` | 优先级变更 |
| `update_curr` | `update_curr_scx` | 记账（§7.1） |
| `uclamp_enabled` | `1` | 启用 uclamp |

> **`migrate_task_rq` 故意不实现**（`ext.c:5071` 注释）：任务到 CPU 的映射是**临时的**，不需要迁移回调。
> **`task_fork`/`task_dead` 也不在方法表**：内核**直接调用** `scx_fork()` / `sched_ext_dead()`，因为需要**所有任务**（不论当前是哪个调度类）的 fork/dead 通知。

### 11.1 DL server：`ext_server`

- 每个 rq 有一个 `rq->ext_server`（`struct sched_dl_entity`），由 `ext_server_init()`（`ext.c:3463`）初始化；
- 它**代表 ext 类去申请一块 DL 带宽**，再由 `ext_server_pick_task()`（`ext.c:3451`）以 `force_scx=true` 选 ext 任务；
- 入队时"第一个任务"启动它（`dl_server_start`），`update_curr_scx()` 里给它记账——**与 fair 类的 `fair_server` 同一套路**（见 [[linux/deadline|DL 笔记]] §6）。

### 11.2 kfunc（89 个 `BTF_ID_FLAGS`）

BPF 侧可调用的核心 kfunc（`ext.c:9060+`）分类：

| 类别 | 代表 |
|---|---|
| DSQ 插入/移动 | `scx_bpf_dsq_insert` / `scx_bpf_dsq_insert_vtime` / `scx_bpf_dsq_move_to_local` |
| DSQ 管理 | `scx_bpf_create_dsq` / `scx_bpf_destroy_dsq` / `scx_bpf_dsq_nr_queued` |
| 选核 | `scx_bpf_select_cpu_dfl` / `scx_bpf_pick_idle_cpu` / `scx_bpf_kick_cpu` |
| 切片/vtime | `scx_bpf_task_set_slice` / `scx_bpf_task_set_dsq_vtime` |
| 查询 | `scx_bpf_task_cpu` / `scx_bpf_cpu_curr` / `scx_bpf_now` / `scx_bpf_events` |
| 退出/报错 | `scx_bpf_exit_bstr` / `scx_bpf_error_bstr` / `scx_bpf_dump_bstr` |
| 子调度器 | `scx_bpf_sub_grant` / `scx_bpf_sub_revoke` / `scx_bpf_sub_dispatch` |

kfunc 按**可调用上下文**分组（`scx_kfunc_ids_enqueue_dispatch` / `dispatch` / `cpu_release` / `unlocked`…），由 BPF 校验器按 `KF_*` 标志限制。

---

## ⚠️ 12. 注意点（踩坑清单）

> [!WARNING] 高频坑
> 1. **CPU 只从 Local DSQ 取任务**：BPF 把任务放进 user/global DSQ 只是"排队"，**必须**再由 `dispatch()` 或直投搬到本地 DSQ，否则任务永远不上 CPU。
> 2. **默认接管全部任务**：没设 `SCX_OPS_SWITCH_PARTIAL` 时，加载后**连普通任务都被接管**，不是只影响 `SCHED_EXT`。
> 3. **切片为 0 会告警并 refill**：BPF 忘了给切片不是崩溃，而是**用 20ms 默认值兜底**并打印一次警告；但反复如此说明调度器有 bug。
> 4. **分派是"可失败"的**：`finish_dispatch()` 靠 qseq 认领，BPF **可以乱分派**，内核**静默丢弃**无效分派——别把 `scx_bpf_dsq_insert()` 当"一定成功"。
> 5. **enqueue 里可以直投**：在 `ops.enqueue()` 里调 `scx_bpf_dsq_insert(SCX_DSQ_LOCAL)` 会走 direct dispatch 快路径，绕过 `QUEUED` 交接；这是**有意设计的优化**。
> 6. **bypass 时不再回调 BPF**：进入 bypass 后内核**不信任** BPF，固定短切片轮转；此时 BPF 的 `tick` 等回调不会被调用。
> 7. **enable 走 RT kthread**：不要以为 enable 在调用者上下文同步完成——它被 offload，否则可能饿死调用线程。
> 8. **`ops_state` 竞争靠 qseq**：读代码时若看到 `finish_dispatch()` "放弃分派"的分支，那是**正确的竞争处理**，不是漏。
> 9. **`migrate_task_rq` 不实现**：任务到 CPU 的映射是临时的，别去找它。
> 10. **子调度器是 cid-form**：实现 `ops.sub_attach` / `ops.sub_detach` **必须**用 cid-form struct_ops（`sched_ext_ops_cid`），否则加载时直接 `scx_error("sub_attach/sub_detach requires cid-form struct_ops")`（`ext.c:7547`）。
> 11. **watchdog 会"归责到 drainer"**：任务卡在别人的 DSQ 上，被踢的是**那个调度器**，不一定是任务属主。

**调试入口**：

- `/sys/kernel/sched_ext/state`、`root/ops`、`root/events`（诊断计数器）、`enable_seq`、`nr_rejected`；
- `SysRq-D` 触发 debug dump（不终止调度器，经 `sched_ext_dump` tracepoint 读取）；`SysRq-S` 终止并回退；
- `scx_bpf_dump()` / `scx_bpf_error()` 从 BPF 侧输出。

---

## 🧭 一句话总结

> **Linux sched_ext = `kernel/sched/ext/` 下一个由 BPF 定义的调度类：`scx_sched` 是调度器实例、`scx_rq` 是每 CPU 状态、`scx_dispatch_q`（DSQ）是核心与 BPF 之间的缓冲；入队时 `scx_do_enqueue_task()` 分流（BPF / 本地 / bypass / global），经 `ops_state` 归属状态机把任务交给 BPF；选队时 `dispatch_one()` 让 BPF 补货、`pick_task_scx()` 只从 Local DSQ 取任务；`finish_dispatch()` 用 qseq 安全提交分派；切片耗尽即重选；bypass + watchdog + `scx_tasks` 全量清单保证"BPF 写错也不崩"。**

---

## 🔗 相关

- [[principle/ext|sched_ext 可扩展调度原理]]：机制/策略分离、事件模型、DSQ 与兜底的理论
- [[linux/cfs|Linux CFS/EEVDF 实现与注意]]：fair 类实现；`ext_server` 与 `fair_server` 同源
- [[linux/deadline|Linux DL 实现与注意]]：DL server（`ext_server` 背后的带宽服务器）
- [[linux/rt|Linux RT 实现与注意]]、[[linux/stop|Linux STOP 实现与注意]]、[[linux/idle|Linux IDLE 实现与注意]]