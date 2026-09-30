---
title: kernel mm视图：32 位内核 1GB 虚拟地址全景与启动六阶段
tags:
  - linux-kernel
  - arm
  - mm
  - mmu
  - virtual-memory
  - memory-map
  - self-booting
  - imx6ull
status: Completed
created: 2026-09-30
---

# 🗺️ kernel mm视图：32 位内核 1GB 虚拟地址全景与启动六阶段

> [!INFO] 笔记概览
> - **定位**：统观 32 位 ARM 内核启动阶段在 `setup_arch()` 里的内存相关流程——从主板识别、早期临时映射，到 `adjust_lowmem_bounds()` 两度计算分界线、建立终身主内核页表，再到 SMP/KVM 核查与控制权交接。
> - **内容范围**：1GB 虚拟地址空间全景布局 · 启动六阶段（板子匹配 / 早期映射 / 内存与页表 / 设备树解包 / SMP 与虚拟化 / 保留与交卷）· 分界线确立代码块与「两次计算」的来龙去脉
> - **推荐路径**：§1 地址空间全景 → §2 启动六阶段全景 → §3 分界线两次计算 → §4 校验与延伸

---

## 🗺️ 1. 1GB 虚拟地址空间全景（0xC0000000 – 0xFFFFFFFF）

```
32 位内核 1GB 虚拟地址空间全景 (0xC0000000 - 0xFFFFFFFF)
┌──────────────────────────────┬────────┬───────────────────┬────────┬──────────┐
│   Direct Mapping (直接映射区)  │ 8MB 隔离│      vmalloc      │ fixmap │ 异常向量表 │
│   [PAGE_OFFSET ~ 物理上限+Δ]   │ (Gap)  │  (含 ioremap 外设) │  固定   │ (Vectors)│
└──────────────────────────────┴────────┴───────────────────┴────────┴──────────┘
▲                              ▲        ▲                   ▲        ▲
0xC0000000                     │        VMALLOC_START       │        0xFFFF0000
                               │                            0xFFF00000
                       【这里就是分界线！】
                    (被 vmalloc/fixmap 顶住)
```

> [!NOTE] 这 1GB 各自归谁
> - **Direct Mapping**：从 `PAGE_OFFSET` 起，映射「物理上限 + Δ」的全部 lowmem，是内核自身、`kmalloc`、`__va/__pa` 的地盘。
> - **vmalloc 区**（含 `ioremap` 外设）：由 `VMALLOC_START` 起步，动态映射、不连续。
> - **fixmap**：固定映射，供 `kmap_atomic` 等短临时映射使用。
> - **异常向量表（Vectors）**：`0xFFF00000` 附近，`0xFFFF0000` 见异常向量。
>
> 三者把 lowmem 的 direct map「顶住」，压出了 §3 里反复计算的 `arm_lowmem_limit` 分界线。

---

## 🚀 2. 启动六阶段全景（`setup_arch()` 内存相关流程）

### 阶段 1：验明正身与主板匹配（Processor & Board Match）

```
   ├── setup_processor()                 ; 读 MIDR，打除法/TLS补丁，挂载 cache/tlb 汇编表
   ├── setup_machine_fdt() / tags        ; 解析设备树 (DTB) 或 ATAGS，识别主板
   └── dump_machine_table()              ; [兜底] 匹配失败时打印支持列表并原地卡死
```

> [!WARNING] 主板匹配失败即终止
> 匹配不到与自身机器号（Machine ID）相符的板子时，`dump_machine_table()` 会打印所有支持列表并**原地卡死**，拒绝继续带病启动。遇到"卡在打板子列表"的现象，多半是 DTB 与内核支持的 machine 表不匹配，先核对板级机器号/设备树版本。

### 阶段 2：早期映射与参数探测（Early Mapping & Params）

```
   ├── early_fixmap_init() / early_ioremap_init()  ; 搭建临时虚拟窗口
   ├── parse_early_param()               ; 解析早期命令行 (包含前面分析的 early_mem "mem=")
   └── setup_dma_zone() / xen / efi      ; DMA 约束与固件接口准备
```

### 阶段 3：物理内存测量与界碑确立（Memory Bounds & Page Tables）

```
   ├── adjust_lowmem_bounds() [第1次]    ; 算 784MB Lowmem 生死线
   ├── arm_memblock_init()               ; 录入物理内存并扣除 kernel 代码/数据等禁区
   ├── adjust_lowmem_bounds() [第2次]    ; 剔除被挖走的空间后，最终锁定 arm_lowmem_limit
   ├── paging_init()                     ; 建立终身主内核页表 (Swapper PGD)，开启完整映射
   └── request_standard_resources()      ; 在 /proc/iomem 树状画地盘 (System RAM, Code, Data)
```

### 阶段 4：设备树解包与电源/重启（DT Unflatten & Reboot）

```
   ├── register_restart_handler()        ; 挂接 arm_restart_nb 通知链
   └── unflatten_device_tree()           ; 将平铺的扁平设备树二进制 (FDT) 解构成树状节点链表
```

### 阶段 5：多核拓扑与虚拟化核查（SMP & Virtualization）

```
   ├── smp_init_cpus()                   ; 发现所有次核心
   ├── smp_build_mpidr_hash()            ; [前面分析的核心] 计算 24位稀疏ID 到紧凑 0~7 的位移压缩
   └── hyp_mode_check()                  ; 核对所有核是否一致在 HYP 模式，决断 KVM 资格
```

### 阶段 6：终局保留与交卷（Reservation & Handoff）

```
   ├── reserve_crashkernel()             ; 按 128MB 对齐从 Lowmem 挖出 Kdump 避难所
   └── mdesc->init_early()               ; 移交控制权给具体 SoC 厂商的早期钩子
```

---

## 📐 3. 分界线 `adjust_lowmem_bounds()` 的两次计算

关键代码块：

```c
adjust_lowmem_bounds();
arm_memblock_init(mdesc);
/* Memory may have been removed so recalculate the bounds. */
adjust_lowmem_bounds();
```

- **第 1 次算**：先根据虚拟地址给 `vmalloc` 留出的天花板，对设备树初报的物理内存做粗测，卡死一个初始的 `arm_lowmem_limit`。

- **中间夹着 `arm_memblock_init`**：在这个函数里，内核会解析设备树保留节点、预留各种 DMA/CMA 空间，甚至可能把某些不可用的物理坏页直接从 `memblock` 移除（Removed）。

- **第 2 次再算**：由于某些处于低端内存边缘的物理内存块可能已经被扣除或移除，导致最高连续物理地址发生了坍塌，因此必须**立刻重新修正一次生死线**，防止后续建页表时访问到空洞。

> [!WARNING] 为什么必须重算第二次
> `arm_memblock_init()` 既可能挖走 DMACMA、又可能剔除坏页，最高连续物理地址随之坍塌。若只用第 1 次的粗测上限去建页表，就可能把 `memblock` 里的空洞也当成映射目标。所以要**在内存被改动后再精算一次**，让生死线与真实内存严密对齐，杜绝访问空洞。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 内存启动流程验证必过
> - [ ] **全景布局**：认清 lowmem direct map、vmalloc、fixmap、Vectors 四段在 1GB 内的相对位置
> - [ ] **分界线定位**：确认 `arm_lowmem_limit` 是被 vmalloc/fixmap 从高地址"顶住"压出来的
> - [ ] **阶段 1**：`setup_processor()` 读 MIDR；主板匹配失败 `dump_machine_table()` 打印并卡死
> - [ ] **阶段 2**：`early_fixmap_init` / `early_ioremap_init` 搭临时窗口；`parse_early_param` 处理 `mem=`
> - [ ] **阶段 3**：两次 `adjust_lowmem_bounds()` 之间夹着 `arm_memblock_init` 扣除禁区；`paging_init()` 建 Swapper PGD
> - [ ] **阶段 4**：`register_restart_handler` 挂 `arm_restart_nb`；`unflatten_device_tree` 解包 FDT
> - [ ] **阶段 5**：`smp_init_cpus` / `smp_build_mpidr_hash` 压缩稀疏 ID；`hyp_mode_check` 决断 KVM 资格
> - [ ] **阶段 6**：`reserve_crashkernel` 按 128MB 对齐挖 Kdump；`mdesc->init_early` 交卷给厂商钩子
> - [ ] **两次重算**：确认第 2 次 `adjust_lowmem_bounds()` 在内存被扣除/移除后执行，页表不越入空洞

---

## 🔗 思考与延伸索引

- [[KERNEL_SET_UP]] — 启动流程主线：head.S 一路到 `start_kernel()` / `setup_arch()`
- [[arm mm init.c]] — `adjust_lowmem_bounds`、`zone_sizes_init`、`swiotlb_init` 的进一步展开
- [[mmu.c kernel起步]] — direct map / vmalloc / PKMAP / FIXMAP 布局的页表级实现
- [[lowmem 与 highmem]] — §3 算出的 `arm_lowmem_limit` 如何切出 low/high 两带并落地成 zone

---

## 📚 参考资料

1. **Linux Kernel 源码** — `arch/arm/kernel/setup.c`（`setup_arch()`、`setup_processor()`、`setup_machine_fdt()`）.
2. **Linux Kernel 源码** — `arch/arm/mm/mmu.c`（`adjust_lowmem_bounds()`、`paging_init()`、`early_fixmap_init`）.
3. **Linux Kernel 源码** — `arch/arm/mm/init.c`（`arm_memblock_init()`、`zone_sizes_init()`、`request_standard_resources()`）.
4. **Linux Kernel 源码** — `arch/arm/kernel/devtree.c`（`unflatten_device_tree()`）、`arch/arm/kernel/smp.c`（`smp_init_cpus`）、`arch/arm/kernel/hyp-stub.c`（`hyp_mode_check()`）.
5. **ARM Architecture Reference Manual** — ARMv7-A 虚拟内存与异常向量（Vectors）段说明.