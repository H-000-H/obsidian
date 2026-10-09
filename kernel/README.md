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

| 项 | 值 |
| --- | --- |
| **内核版本** | **Linux 7.3.0-rc6**（mainline，`torvalds/linux` master） |
| **源码树路径** | `/home/h-000-h/project/linux-source-code` |
| **树 HEAD** | `7b63ef2d5`（2026-10-07，`Merge tag 'for-7.3-rc6-tag'`） |
| **平台重点** | ARM32（i.MX6ULL / Cortex-A7）为主，兼有通用子系统 |

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
