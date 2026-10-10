---
title: kobject ↔ sysfs ↔ kernfs 映射与分层
aliases:
  - kobject
  - kset
  - Linux 设备模型
  - sysfs 目录生成
  - kobj_ns 命名空间
tags:
  - linux/kernel
  - driver-model
  - kobject
  - kset
  - sysfs
  - kernfs
  - device-model
  - namespace
status: Completed
created: 2026-10-10
kernel: "7.3.0-rc6 (mainline)"
arch: generic
---

# 🧩 设备模型：kobject ↔ sysfs ↔ kernfs 映射与分层

> [!INFO] 笔记概览
> - **定位**：梳理 Linux 设备模型的**地基**——`kobject` 作为"原子"如何经 `sysfs` 映射成 `kernfs` 目录树，以及 `kset` / `device` / `class` / `bus` 如何在此之上叠语义。它是理解 `/sys`、`/dev`、udev 与容器命名空间隔离的前置。
> - **内容范围**：三层全景 · `kobject` 原子与生命周期 · 嵌入 vs 裸（`ktype->release`）· `kobject ↔ kernfs_node` 双向链接 · 目录生成（parent vs kset）· 顶层组织者四变体 · sysfs 命名空间（`kobj_ns`）· `/sys` vs `/dev` 两张脸 · 真机实例
> - **推荐路径**：§1 三层全景 → §2 `kobject` 原子 → §4 映射双向链接 → §5 目录生成（核心）→ §6 组织者变体 → §7 命名空间 → §8 `/sys` vs `/dev` → §11 Checklist
> - 本文所有结论均对照本仓库源码核实，格式为 `文件:行号`。

相关笔记：[[init.main.c]] · [[ARM Kernel head.S 内核启动 学习笔记]] · [[kernel mm视图]]

---

## 🧭 1. 三层全景（一张图）

```
┌─ /sys (VFS)
│     ▲ 真实挂载处 = kernfs_node 树（结构核心）
│     │   kn->priv → kobject      kn->ns → 命名空间标签
│
│  sysfs 适配层：kobject ↔ kernfs_node 桥（sd / priv 双向链接、
│                 create_dir / remove_dir、sysfs_ops 路由）
│
▼  设备模型层：kobject / kset / kref / parent / ktype
    —— 内核里的"真相"，带生命周期，是各种具体功能
```

> [!IMPORTANT] 一句话主线
> **`kernfs_node` 是 `/sys` 的结构核心（真实长在 /sys 里的树，kernfs 为 sysfs/cgroup 共用的通用伪文件系统）；`kobject/kset/kref/parent/ktype` 是内核侧的"设备模型层"（对象的真相与生命周期）；`sysfs` 是二者之间的桥，把 `kobject` 经 `sd`/`priv` 映射成 `kernfs_node` 并路由读写。`kobject_add` 本身不现身 `/sys`，而是触发 kernfs 造出带 ns 标签、指回自身的 `kernfs_node`——那才是真实挂载处。**

---

## ⚛️ 2. `kobject` 是原子：自带完整生命周期

`kobject` 是一个**有完整、自管理生命周期的一等公民对象**，不依赖 device/class/bus 任何高层概念。它由一组状态位 + 引用计数驱动（`lib/kobject.c`）：

| 阶段 | 触发函数 | 状态变化 |
| --- | --- | --- |
| 出生 | `kobject_init` | `kref=1`、`state_initialized=1` |
| 进 sysfs | `kobject_add` | `kobj_kset_join`、`state_in_sysfs=1` |
| 改名 | `kobject_rename` | 换 `name` + 移 sysfs 节点 + `KOBJ_MOVE` |
| 删表现 | `kobject_del` / `__kobject_del` | 删目录/文件、退出 kset、`state_in_sysfs=0`、补 `KOBJ_REMOVE` |
| 放手 | `kobject_put` | `kref_put`，**归零才**继续 |
| 死 | `kobject_release` → `kobject_cleanup` | 调 `ktype->release`、释放名字、还父引用 |

那几个 `state_*` 标志（`state_initialized` / `state_in_sysfs` / `state_add_uevent_sent` / `state_remove_uevent_sent`）就是 `kobject` **自己生命周期的记账字段**。

**它和"上层大结构"唯一的耦合点，就是 `ktype->release` 这一个回调**——这正是"裸 vs 嵌入"两种情况的真正分水岭。

---

## 🔌 3. 嵌入 vs 裸：唯一接缝 `ktype->release`

| 形态 | 谁提供内存 | `release` 干啥 | 例子 |
| --- | --- | --- | --- |
| **裸 kobject** | `kobject_create_and_add` 动态分配 | `kfree(kobj)` 自了 | `/sys/kernel`、`/sys/kernel/mm/ksm` |
| **嵌入 kobject** | 外层结构体已含 kobject | `kfree(container_of(...))` 顺带释放**外层** | `struct device` / `struct kset` |

- **嵌入**：`struct device` 内嵌 `struct kobject kobj;`（`net_device` 再内嵌 `device`）；计数归零触发 `device_release`，用 `container_of` 找回外层整体释放。
- **裸**：`dynamic_kobj_ktype`，`release` 直接 `kfree(kobj)`。

> [!NOTE] "kobject 有生命周期"与"device 有生命周期"不冲突
> 它们是**同一个对象**，只是 `kobject` 是那段"被计数、被挂 sysfs、被追踪状态"的可见/可管理部分；归零时通过 `release` 这个接缝，把"释放整个外层对象"交给上层办。裸 kobject 则在 `release` 里把 `kobj` 本身 `kfree` 掉。

---

## 🔗 4. `kobject ↔ kernfs_node` 双向链接（模型侧 vs 视图侧）

`sysfs_create_dir_ns`（`fs/sysfs/dir.c:40`）一次建立双向链接：

```c
kn = kernfs_create_dir_ns(parent, kobject_name(kobj), 0755, uid, gid, kobj, ns);
//                                                              ^^^^ priv=kobj   ^^ ns标签
kobj->sd = kn;                 // kobject → kernfs_node（模型指向视图）
```

| 对象 | 是什么 | 在哪 |
| --- | --- | --- |
| `struct kobject`（设备模型侧） | 内核内存里的对象模型 | 内核内存，**不在 /sys 里** |
| `struct kernfs_node`（sysfs 侧） | /sys 树里的真实目录节点 | **/sys 里** |

- `kobj->sd = kn`：kobject 指向它的 sysfs 节点；
- `kn->priv = kobj`：sysfs 节点指回 kobject（第 6 参 `kobj`）；
- `kn->ns = ns`：节点携带命名空间标签（`include/linux/kernfs.h:216`）。

> [!IMPORTANT] 不是"上下两层"，是两个对象双向指
> 严格讲只有 `kernfs_node` 真正"在 `/sys` 里"；`kobject` 的目录之所以出现，是因为 add 时为它造了个带标签、指回自身的 `kernfs_node`。模型侧与视图侧是**同一回事的两端**。

网卡全链：`struct net_device → struct device → struct kobject → (sd) → struct kernfs_node(带 ns) → priv → 回到 kobject`。

---

## 🌳 5. 目录生成：parent vs kset（核心）

目录的**最终落点永远只有一条规则**（`fs/sysfs/dir.c:49`）：

```c
if (kobj->parent)
    parent = kobj->parent->sd;     // 目录建在 parent 的 kernfs_node 下
else
    parent = sysfs_root_kn;        // 否则建在 /sys 根
```

`sysfs_create_dir_ns` **只认 `kobj->parent`，根本不碰 kset**。kset 影响位置靠 `kobject_add_internal`（`lib/kobject.c:257`）先把 kset **升格**为 parent：

```c
parent = kobject_get(kobj->parent);
/* join kset if set, use it as parent if we do not already have one */
if (kobj->kset) {
    if (!parent)
        parent = kobject_get(&kobj->kset->kobj);  // ← kset 的 kobj "升格"成 parent
    kobj_kset_join(kobj);                          // ← 仅做分组（挂进 kset 成员链表）
    kobj->parent = parent;
}
error = create_dir(kobj);
```

**规则表：**

| 你给的 | 目录落到哪 | kset 额外干啥 |
| --- | --- | --- |
| **显式 `parent`** | `parent->sd` 之下 | —（若同时设 kset，仅分组） |
| **只设 `kset`，无 parent** | `kset->kobj.sd` 之下（升格为 parent） | `kobj_kset_join` 收进成员链表 |
| **两者都不设** | `/sys` 根（`sysfs_root_kn`） | — |
| **两者都设** | **显式 parent 赢**（位置）；kset **只分组** | `kobj_kset_join` |

> [!WARNING] kset 不是"另一种生目录机制"
> 它和 parent 走的是**同一条 `kobj->parent->sd` 路**——`kset` 只是"你懒得写 parent 时我替你可当 parent"，多出来的价值是 `kobj_kset_join` 的**分组/迭代能力**。一句话：**parent 只管"在哪"，kset 管"在哪 + 归到哪一组"。**

顺带一个引用计数细节：`parent = kobject_get(...)` 给 parent **多加一个引用**，保证父目录在子目录存活期间不被释放；失败时 `kobject_put(parent)` 还回。

---

## 🗂️ 6. 顶层组织者全是 `kobject/kset` 变体

从 `kobject` 原子视角，设备模型的顶层组织者**全是"kobject/kset + 不同语义"**，没有谁比谁更特殊：

```
kobject（最底层原子：kref + state_* + sysfs 表现）
├─ 嵌入 device → /sys/devices 真身；被 class/bus 用 symlink 映射成视图
├─ kset        → 容器目录（/sys/class 顶层、/sys/bus、/sys/devices…）
└─ class(bus)  → 经 subsys_private 内含 kset（自身 /sys/class/<name> 目录）
                  + devices/drivers 子 kset，且持 ns_type/namespace
                  → 给其下设备节点贴 netns 标签、实现容器视图隔离
```

**`class` 的精确结构**（`struct class` 不直接内嵌裸 kobject）：

```c
/* include/linux/device/class.h:50 */
struct class {
    const char *name;
    ...
    const struct kobj_ns_type_operations *ns_type;      // :64
    const struct ns_common *(*namespace)(const struct device *dev);  // :65
    ...
};
/* drivers/base/base.h:43 */
struct subsys_private {
    struct kset subsys;        // ← class 自身目录 /sys/class/<name>
    struct kset *devices_kset; // ← /sys/class/<c>/devices
    struct kset *drivers_kset; // ← /sys/class/<c>/drivers
    ...
};
```

`class_register` 末尾用 `kset_register(&cp->subsys)` 把 `/sys/class/<name>` 立起来——**就是 kset 那套机制**。

**kset 三分惯例**（与 `kobject` 同构）：

| kset 层 | kobject 层 | 含义 |
| --- | --- | --- |
| `kset_init(k)` `:855` | `kobject_init` | 内存初始化（无分配/无 sysfs） |
| `kset_create(...)` `:1005` | `kobject_create` | 动态分配一个对象 |
| `kset_register(k)` `:904` | `kobject_init_and_add` | init + 加 sysfs（+ 自动发 `KOBJ_ADD`） |
| `kset_create_and_add(...)` `:1048` | — | create + register 一步 |

> [!NOTE] `kset_register` 比 `kobject_add` 多一步
> `kobject_add` 故意**不发** `KOBJ_ADD`，把 uevent 时机留给上层（防 udev 抢跑）；`kset_register` 因 kset 注册即就绪，直接在末尾 `kobject_uevent(KOBJ_ADD)`——是那条规则的"便捷包装例外"。

---

## 🌐 7. sysfs 命名空间（`kobj_ns`）

sysfs 的 ns 框架是"**能力门禁 + 引用取/还**"双轨，核心是一张按类型索引的分发表：

```c
/* include/linux/kobject_ns.h:27 */
enum kobj_ns_type {
    KOBJ_NS_TYPE_NONE = 0,
    KOBJ_NS_TYPE_NET,        // ← 本树实际只有"网络命名空间"这一种
    KOBJ_NS_TYPES
};
/* lib/kobject.c:1069 */
static const struct kobj_ns_type_operations *kobj_ns_ops_tbl[KOBJ_NS_TYPES];
```

`kobj_ns_type_register`（`lib/kobject.c`）把某类型后端存进表：**合法性校验 → 防重复(-EBUSY) → 存入**。注册者仅 `net_ns_type_operations`（`net/core/net-sysfs.c:2413`）。

**三个关键函数：**

| 函数 | 本质 | 是否取引用 |
| --- | --- | --- |
| `kobj_ns_current_may_mount` | **门禁**：调 `current_may_mount()` 返 bool | ❌ 只看不拿 |
| `kobj_ns_grab_current` | **真实取引用**：`grab_current_ns()`（`get_net`）返 `ns_common *` | ✅ 拿到 +1 引用 |
| `kobj_ns_drop` | **真实还引用**：`drop_ns()`（`put_net`） | ✅ 还回 |

`NET` 的门禁实现（`net/core/net-sysfs.c:2160`）：

```c
static bool net_current_may_mount(void) {
    struct net *net = current->nsproxy->net_ns;
    return ns_capable(net->user_ns, CAP_SYS_ADMIN);   // 查当前 netns 所属 userns 的权限
}
```

> [!IMPORTANT] 门禁只判断准入，显隐靠后面两个
> `may_mount` 是挂载边界的一次性准入策略（查 `CAP_SYS_ADMIN`），**不取引用、不参与显隐**。显隐链是：挂载用 `kobj_ns_grab_current` 抓当前 netns 存为挂载的 `ns_tag`（`fs/sysfs/mount.c:55`，卸载时 `:31/:75` 用 `kobj_ns_drop` 还）→ kernfs 拿每个节点 `ns` 标签与 `ns_tag` 比对决定显隐；节点自身也持 ns 引用。

**"两个引用计数"**：`kobject` 的 `kref`（`kobject_get/put`）与命名空间的引用计数（`grab_current_ns/drop_ns`）是**两个相互独立的计数**，一个带标签的 sysfs 挂载/节点会同时握着两者。

`net_class` 的 ns 标签来源（`net/core/net-sysfs.c:1163`）：

```c
if (dev->class && dev->class->namespace)
    return dev->class->namespace(dev);   // ← 设备目录的 ns 标签取自它所属 class 的回调
```

> [!NOTE] class 兼"ns 标签源"
> `struct class` 带 `ns_type` + `namespace`（`class.h:64-65`），`class_register` 强制成对（`drivers/base/class.c:186-195`）。`net_class` 据此使 `/sys/class/net` 打上 netns 标签、按容器过滤——这就是容器里 `ls /sys/class/net` 看到网卡变少的根因。

---

## 🗃️ 8. 两张脸：`/sys`（元数据） vs `/dev`（I/O 节点）

| | `/sys/class/<c>/<device>` | `/dev/<node>` |
| --- | --- | --- |
| 本质 | 设备模型/元数据视图（kernfs 上，属性 + symlink） | **设备特殊文件**（字符/块，带 major:minor） |
| 干嘛用 | **查看/配置**：读 `address`、`mtu`，写 `power/control`，绑/解绑驱动 | **真正 I/O**：`open/read/write/ioctl` |
| 怎么来 | `kobject_add`/`device_add` → sysfs 目录（真身在 `/sys/devices`） | **udev** 监听 uevent 后 `mknod` |

桥就在 `device_add`（`drivers/base/core.c`）：

```c
3727:	device_create_file(dev, &dev_attr_dev);   // 造 "dev" 文件，内容 "MAJOR:MINOR"
3742:	kobject_uevent(&dev->kobj, KOBJ_ADD);     // 广播 ADD 事件
```

- `KOBJ_ADD` uevent → **udev** 读到 → 去 sysfs 读该 `dev` 文件拿 `major:minor` → 在 `/dev` `mknod` 出节点。
- **sysfs 是真相源，`/dev` 是它用户态镜像**，桥 = 设备号 + uevent。
- sysfs 自带的按号索引：`/sys/dev/{char,block}/<major>:<minor>`（`drivers/base/core.c:2405/2408`）。

> [!WARNING] 网络设备没有 `/dev` 节点
> `/sys/class/net/eth0` 通常**没有**对应 `/dev` 节点——网络走 socket 而非字符设备。所以网卡"真实使用"既不靠 `/dev`（压根没有），也不靠 `open(/sys/class/net/...)`（那是元数据），而是靠网络栈；`/sys/class/net/` 下的属性用于**配置与状态查询**。

---

## 🔬 9. 真机实例

| 路径 | 类型 | 说明 |
| --- | --- | --- |
| `/sys/kernel`、`/sys/kernel/mm` | **裸 kobject** | `kernel_kobj = kobject_create_and_add("kernel", NULL)`（`kernel/ksysfs.c:221`）、`mm_kobj = kobject_create_and_add("mm", kernel_kobj)`（`mm/mm_init.c:243`）；parent/kset 皆 NULL → 落 `/sys` 根 |
| `/sys/kernel/mm/ksm/advisor_min_pages_to_scan` | **裸 kobject + 属性** | 文件不是"存值的抽屉"，而是 `kobj_attribute` 活窗口：`cat` 触发 `.show` 读出内核变量，`echo` 触发 `.store` 写回（`mm/ksm.c`） |
| `/sys/kernel/mm/transparent_hugepage/hugepages-<size>kB` | **嵌入式 kobject**（mTHP） | `thpsize_create` 用 `kobject_init_and_add` 为每个 order 建目录（`mm/huge_memory.c`）；`size = PAGE_SIZE<<order` |
| `/sys/class/net/{enp55s0,lo,wlo1}` | **class + ns** | class 目录下是 symlink 指回 `/sys/devices/...` 真身；带 netns 标签（容器里过滤） |
| `/dev/...` | **设备节点** | udev 据 uevent + `dev` 属性造出，供 `open/read/write` |

> [!TIP] 一个 `cat` 走完整链
> `cat /sys/kernel/mm/ksm/advisor_min_pages_to_scan` → VFS 读 sysfs 文件 → `kobj_sysfs_ops.show`（`kobj_attr_show`）→ `container_of_const` 找回 `kobj_attribute` → 调属性自己的 `.show` → 读内核变量 → 打印。这就是"裸 kobject + 属性窗口"机制的落地。

---

## 📖 10. 常见误区对照

| 常见口径 | 实际 | 证据 |
| --- | --- | --- |
| "`/sys` 目录是 VFS 目录" | 是 **kernfs** 目录（`kernfs_node`），kernfs 再对接 VFS | `fs/kernfs/`、`fs/sysfs/dir.c` |
| "kset 是另一种生目录机制" | 走**同一条** `kobj->parent->sd`；kset 被升格为 parent + 分组 | `lib/kobject.c:260-265` |
| "class 是独立的非 device 对象" | class 经 `subsys_private` 内含 **kset**；其下设备是 **symlink** 视图 | `drivers/base/base.h:43` |
| "`kobject_add` 会发 uevent" | 故意**不发**，留给上层；`kset_register` 便捷地代发 | `lib/kobject.c` |
| "`/sys/class` 里的设备就是真身" | 是 symlink，真身在 `/sys/devices/...` | `class.c:607` `sysfs_create_link` |
| "网卡能在 `/dev` 找到" | 网络设备**无** `/dev` 节点，走 socket | net 子系统 |
| "grab 会顺便检查挂载权限" | grab 是**无策略原语**，准入由 `may_mount` 在挂载处单独查 | `fs/sysfs/mount.c:47/55` |
| "`hpage_pmd_size` 是 x86 专属" | 4KB 基页下 x86-64 与 arm64 的 PMD 块都是 2MB | `include/linux/huge_mm.h` |

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 设备模型地基验证必过
> - [ ] **三层**：能说清 kernfs_node(结构核心) / sysfs(桥) / kobject(设备模型原子) 的职责
> - [ ] **生命周期**：背出 `init→add→rename→del→put→release→cleanup` 与 `state_*` 记账字段
> - [ ] **嵌入 vs 裸**：解释 `ktype->release` 是唯一接缝（`container_of` 释放外层 vs `kfree(kobj)`）
> - [ ] **双向链接**：说出 `kobj->sd` / `kn->priv` / `kn->ns` 三者关系与来源
> - [ ] **目录生成**：讲清 `kobject_add_internal` 把 kset 升格为 parent、`sysfs_create_dir_ns` 只认 `parent->sd`/root，及四种组合规则
> - [ ] **组织者变体**：说明 kset/device/class/bus 都是 kobject/kset + 语义，class 经 `subsys_private` 内含 kset
> - [ ] **命名空间**：区分 `may_mount`(门禁) 与 `grab/drop`(真实取还引用)；知 ns 标签取自 `class->namespace()`
> - [ ] **两张脸**：说清 `/sys`(元数据) vs `/dev`(I/O) 及桥(`KOBJ_ADD` uevent + `dev` 属性 + udev)
> - [ ] **真机对号**：能在 `/sys/kernel/mm/ksm`(裸)、`/sys/class/net`(class+ns)、`/dev`(节点) 上指认原理

---

## 🔗 11. 相关笔记

- [[init.main.c]] — `driver_init()`/`ksysfs_init()` 在 `do_basic_setup` 里建立设备模型与 `/sys/kernel` 地基
- [[ARM Kernel head.S 内核启动 学习笔记]] — 汇编视角：内核如何走到 C 世界，`start_kernel` 后才逐步建立子系统
- [[kernel mm视图]] — 内存/地址空间布局，理解 sysfs/kernfs 所在的内核内存底座
- [[kernel 中断系统（ARM32）学习笔记]] — uevent/`kworker`/软中断等异步机制的下半部底座

---

## 📚 参考资料

1. **Linux Kernel 源码（本仓库 7.3.0-rc6）** — `lib/kobject.c`（`kobject_add_internal` `:242`、kset 四函数 `:855`/`:1005`/`:904`/`:1048`、`kobj_ns_ops_tbl` `:1069`）.
2. **Linux Kernel 源码（本仓库）** — `fs/sysfs/dir.c`（`sysfs_create_dir_ns` `:40`）、`fs/sysfs/mount.c`（ns 挂载 `:31/47/55/75`）.
3. **Linux Kernel 源码（本仓库）** — `include/linux/kobject.h`（`struct kobject`、`sd` 字段）、`include/linux/kernfs.h`（`kernfs_node.priv`/`.ns` `:216`）、`include/linux/kobject_ns.h`（`enum kobj_ns_type` `:27`、`kobj_ns_type_operations` `:40`）.
4. **Linux Kernel 源码（本仓库）** — `drivers/base/class.h`（`struct class` `:50`、`ns_type`/`namespace` `:64-65`）、`drivers/base/base.h`（`subsys_private` `:43`）、`drivers/base/class.c`（class ns 校验 `:186-195`）、`drivers/base/core.c`（`dev` 属性 `:3727`、`KOBJ_ADD` `:3742`）.
5. **Linux Kernel 源码（本仓库）** — `net/core/net-sysfs.c`（`net_current_may_mount` `:2160`、`net_ns_type_operations` `:2188`、`net_class` `:2249`、注册 `:2413`）.

（注：本文为源码精读笔记，事实来源以本仓库逐行核实为准，格式 `文件:行号` 高频内联。）
