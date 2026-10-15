---
title: "kernel 学习笔记索引与版本基线"
aliases:
  - kernel 笔记索引
  - kernel README
tags:
  - linux/kernel
  - meta
  - index
status: Completed
created: 2026-10-09
kernel: "7.3.0-rc6 (mainline)"
---

# 📚 kernel 学习笔记索引

> 本目录是 **Linux 内核源码精读笔记**集合。所有结论均以**源码逐行核实**为准，正文引用格式统一为 `文件:行号`。

---

## 🧷 版本基线（重要）

| 项          | 值                                                     |
| ---------- | ----------------------------------------------------- |
| **内核版本**   | **Linux 7.3.0-rc6**（mainline，`torvalds/linux` master） |
| **源码树路径**  | `/home/h-000-h/project/linux-source-code`             |
| **树 HEAD** | `7b63ef2d5`（2026-10-07，`Merge tag 'for-7.3-rc6-tag'`） |
| **平台重点**   | ARM32（i.MX6ULL / Cortex-A7）为主，兼有通用子系统                 |

> [!IMPORTANT] 全库版本基线：7.3.0-rc6
> 所有笔记的结论与 `文件:行号` 引用均以 **Linux 7.3.0-rc6（mainline）** 为准，并已逐条对照该源码树核实。

> [!NOTE] 关于 `init/main.c` 的行号
> 本树 `init/main.c` 带**内联中文批注**，因此行号以本树为准；笔记中所有 `init/main.c:行号` **均指本树（已批注）**。

---

## 🗂️ 笔记索引

### 🚀 启动与汇编

| 笔记 | 主源码 | 主题 |
| --- | --- | --- |
| [[ARM zImage head.S 启动 学习笔记]] | `arch/arm/boot/compressed/head.S` | zImage 自解压壳：PIC、落点、DTB、relocate、cache |
| [[ARM Kernel head.S 内核启动 学习笔记]] | `arch/arm/kernel/head.S` | 内核解压后建页表、开 MMU/Cache、跳 `start_kernel` |
| [[KERNEL SET_UP]] | `arch/arm/kernel/setup.c` 等 | `setup_arch()` 架构初始化全景：31 步、768MB 生死线、`paging_init` |
| [[init.main.c]] | `init/main.c` | `start_kernel` → 用户态 init 主线：命令行、initcall 八级、`rest_init`、`kernel_init` |
| [[常用寄存器]] | — | ARM CP15 寄存器速查（SCTLR 等） |
| [[arm 汇编 补充]] | — | ARM 启动路径汇编指令/伪指令速查 |

### 🧠 内存管理

| 笔记 | 主源码 | 主题 |
| --- | --- | --- |
| [[kernel mm视图]] | — | 32 位内核 1GB 虚拟地址全景与启动六阶段 |
| [[lowmem 与 highmem]] | `arch/arm/mm/*` | 32 位内核的 lowmem/highmem 地址分界（含 kmap/swiotlb） |
| [[mmu.c kernel起步]] | `arch/arm/mm/mmu.c` | `mmu.c` 内核起步期的地址空间施工 |
| [[页表预分配 mmu.c]] | `arch/arm/mm/mmu.c` | 页表预分配缓存策略速查表 |
| [[arm mm init.c]] | `arch/arm/mm/init.c` | ARM `mm/init.c` 内存初始化完整手册（memblock/zone/RWX） |

### ⚡ 中断

| 笔记 | 主源码 | 主题 |
| --- | --- | --- |
| [[kernel 中断系统（ARM32）学习笔记]] | `kernel/irq/*`、`arch/arm/kernel/irq.c` | ARM32 中断系统：irq_desc、softirq、tasklet、workqueue、IPI |

### 🧱 设备模型与模块

> [!TIP] 一条主线读所有篇
> Linux 设备/驱动体系翻来覆去就是**四种零件**在组合：**① 描述符/记录**（`struct device`/`cdev`/`module`/`class`/`bus_type`，只登记不干活）· **② 回调/钩子**（`file_operations`/`probe`/`remove`/PM `ops`/module `init`-`exit`，真正干活）· **③ 引擎/管理**（`bus_type` 撮合、`kobj_map`/`cdev_map` 查号、PM core、module loader，把①+②串起来）· **④ 投影到文件**（sysfs/`/sys/class`/`/sys/bus`/属性/udev 建的 `/dev`，给用户态看）。
> 另一切分：**记录层/前台**（device/bus/class，登记+配对+投影）vs **I/O 层/后台**（cdev+`file_operations`，数据进出）。看任何子系统（usb/i2c/net/mtd…）都是这四件套换皮。

| 笔记 | 主源码 | 主题 |
| --- | --- | --- |
| [[kobject-sysfs-kernfs映射分层]] | `lib/kobject.c`、`fs/sysfs`、`fs/kernfs` | 设备模型地基：kobject ↔ sysfs ↔ kernfs 映射与分层 |
| [[sysfs 目录↔结构体字段对照树]] | 本机 `/sys`、`include/linux/{device*.h,kernfs.h,sysfs.h}`、`blkdev.h`、`module.h` | 在**本机真实 `/sys` 目录**上做对照：每目录 = 完整结构体 + 真实目录 + 条目↔字段表；外加属性五分类、kernfs 三节点、命名组=子目录 |
| [[kobject 与 module_kobject 结构精读]] | `include/linux/kobject.h`、`include/linux/module.h` | kobject/kobj_type/kset/attribute 与 module_kobject/module_attribute |
| [[struct device 精读]] | `include/linux/device.h` | 设备通用抽象：模型/总线/驱动/PM/DMA/IOMMU/固件节点 |
| [[struct bus_type 精读]] | `include/linux/device/bus.h`、`drivers/base/bus.c` | 总线撮合模板：match→probe、模板 vs 实例、IRQ 亲和性 |
| [[struct platform_driver 与 probe 时序]] | `include/linux/{init.h,device/driver.h,platform_device.h}`、`init/main.c`、`drivers/base/dd.c`、`drivers/base/platform.c` | 驱动侧另一半：initcall 分级(module_init=level 6→do_initcalls)、device_driver/platform_driver/driver_private、匹配表、两条路径汇合 really_probe、WRITE_ONCE 锁纪律、driver_bound、deferred probe、test_remove/devres 不变式 |
| [[struct class 与设备自动发现]] | `include/linux/device/class.h`、`drivers/base/base.h` | class 视图 + subsys_private + uevent/udev 自动建 /dev 节点、三种"自动"机制 |
| [[驱动实战：从裸模块到 platform_driver]] | 本机实测（Ubuntu `7.0.0-38-generic`） | **动手实践**：手写驱动 4 关递进（裸模块 → 字符设备+/dev → 多实例 → platform_driver+假设备）；kbuild 多目录、10 个真机踩坑集、调试工具箱 |
| [[运行时电源管理 runtime PM]] | `include/linux/pm.h`、`drivers/base/power/` | runtime PM：dev_pm_info / /sys/.../power/ / dev_pm_ops / dev_pm_domain(genpd) |
| [[platform-resource-mfd 结构精读]] | `include/linux/platform_device.h`、`include/linux/ioport.h`、`include/linux/mfd/core.h` | platform 记录层：resource / platform_device / mfd_cell |
| [[struct module 精读]] | `include/linux/module.h`、`kernel/module/*` | 模块子系统：struct module 全字段（导出/holders/许可证/ELF 段） |
| [[file_operations 与字符设备结构精读]] | `include/linux/fs.h`、`include/linux/cdev.h`、`drivers/base/map.c` | 文件 I/O 层：file_operations / cdev / kobj_map（dev_t→cdev 注册表） |

### ⚖️ 调度算法

分两套：`linux/` 讲**内核真实实现**，`principle/` 讲**算法原理**。两者均基于 **Linux 7.3.0-rc6**。

| 主题 | 实现 | 原理 |
| --- | --- | --- |
| CFS / EEVDF | [[linux/cfs]] | [[principle/cfs]] |
| Deadline | [[linux/deadline]] | [[principle/deadline]] |
| sched_ext | [[linux/ext]] | [[principle/ext]] |
| Idle | [[linux/idle]] | [[principle/idle]] |
| RT | [[linux/rt]] | [[principle/rt]] |
| Stop | [[linux/stop]] | [[principle/stop]] |

> 调度笔记的源码基线为 `torvalds/linux` master **7.3.0-rc6**，各篇 frontmatter 内注明了所依据的 commit（如 `22430ae5d90a` / `69f80fef3153`，2026-10-06）。

---

## 📐 笔记约定

- **frontmatter**：`title` / `tags` / `status` / `created`；源码类笔记额外注明版本与基线。
- **标注块**：使用 Obsidian callout（`> [!INFO]` / `[!TIP]` / `[!WARNING]` / `[!CAUTION]` / `[!DANGER]` / `[!IMPORTANT]` / `[!NOTE]` / `[!SUCCESS]`）。
- **图**：流程/时序优先用 `mermaid`。
- **引用格式**：一律 `文件:行号`（源码树以 §版本基线 为准）。
- **状态**：`status: Completed`（已完成）/ `In-progress`（进行中）。

---

## 🧭 待建 / 相关

- `KERNEL_READING_ORDER` —— 被 [[KERNEL SET_UP]] 等引用，**尚未创建**（规划中的"内核源码阅读顺序总纲"）。

---

## 📄 许可

本 vault 内容为个人学习笔记，版权与使用限制见仓库根 `LICENSE.md`（个人使用，未经许可不得复制/再分发）。
