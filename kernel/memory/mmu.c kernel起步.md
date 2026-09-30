---
title: mmu.c 内核起步期的地址空间施工
tags:
  - Kernel
  - ARM
  - MMU
  - Memory-Init
  - Page-Table
  - IMX6ULL
status: In-progress
created: 2026-09-30
---

# 🚧 mmu.c 内核起步期的地址空间施工

> [!INFO] 笔记概览
> - **定位**：ARM 架构层的"地址空间施工文件"——在启动早期把内核的虚拟地址（VA）布局与 VA→PA 页表搭起来，交给正式 MM 体系接手。它**不实现页框分配器**（Buddy 在 `mm/page_alloc.c`）。
> - **内容范围**：真实调用时序 · VA 布局（PAGE_OFFSET/VMALLOC/PKMAP/FIXMAP）· 各函数逐层拆解（`adjust_lowmem_bounds`/`prepare_page_table`/`map_lowmem`/`map_kernel`/`early_fixmap_shutdown`/`devicemaps_init`/`kmap_init`/`early_paging_init`/`set_ptes`/`build_mem_type_table`）· 属性/粒度速查 · 启动总图
> - **推荐路径**：§1 一句话定位 → §2 调用时序 → §3 VA 布局 → §4 各函数拆解 → §5 属性速查 → §6 总图

---

## 🎯 1. 一句话定位与心脏函数

ARM 架构层的**地址空间施工文件**：在启动早期把内核的 VA 布局与 VA→PA 页表搭起来，交给正式 MM 体系接手。它不实现页框分配器（Buddy 在 `mm/page_alloc.c`）。

`create_mapping(md)` 是本文的心脏函数，整条映射链走到最底层，最终把页表写进硬件的 PTE 寄存器：

```
create_mapping(md)
    │
    ├─ 检查 VA 是否允许放这里
    │
    └─ __create_mapping(&init_mm, md(内存描述符), early_alloc, false)
            │
            ▼
           PGD
            │
            ▼
           P4D
            │
            ▼
           PUD
            │
            ▼
           PMD
          /    \
    Section     PTE
                  │
                 4KB
```

---

## ⏱️ 2. 真实调用时序

`adjust_lowmem_bounds()` **不在** `paging_init()` 内部，而是在 `arm_memblock_init()` 前后各被调用一次（预留后内存可能被移除，需重算边界）。以下为 `mmu.c` 中的真实函数与行号：

```
head.S ──► swapper_pg_dir 恒等映射 + 开 MMU（不在本文件，是最初的"临时脚手架"）
                          │
start_kernel ──► setup_arch()
[arch/arm/kernel/setup.c]
   │
   ├─ early_fixmap_init()              mmu.c:396
   │     启用早期 fixmap（ioremap 早期 PTE 池）
   ├─ parse_early_param()
   ├─ early_mm_init()                  mmu.c:1818
   │     ├─ build_mem_type_table()     mmu.c:463   按 CPU/命令行修正 mem_type 属性表
   │     └─ early_paging_init()        mmu.c:1654  ARM_PV_FIXUP：LPAE 下重建启动页表、
   │                                           修正 __pv_offset / __pv_phys_pfn_offset
   ├─ adjust_lowmem_bounds()           mmu.c:1210  第一次算 lowmem_limit
   ├─ arm_memblock_init()              mm/init.c:185  注册 kernel/initrd/页表等到 memblock
   │     └─ arm_mm_memblock_reserve()  mmu.c:1367  预留 swapper_pg_dir
   ├─ adjust_lowmem_bounds()                       第二次（memblock 删减后重算）
   ├─ early_ioremap_reset()
   │
   └─ paging_init()                    mmu.c:1780  ← 总调度
         ├─ prepare_page_table()       mmu.c:1305
         ├─ map_lowmem()               mmu.c:1502
         ├─ memblock_set_current_limit(arm_lowmem_limit)   ← 此后 early_alloc 才安全
         ├─ map_kernel()               mmu.c:1585
         ├─ dma_contiguous_remap()     [dma-mm.c]  CMA
         ├─ early_fixmap_shutdown()    mmu.c:1747
         ├─ devicemaps_init()          mmu.c:1391
         ├─ kmap_init()                mmu.c:1491
         ├─ tcm_init()
         ├─ early_alloc(PAGE_SIZE)                 zero page
         └─ bootmem_init()             mm/init.c:208  ← 名字是历史遗留，现在做的是：
               ├─ sparse_init()                    mem_map 数组
               └─ zone_sizes_init()                free_area_init → per-zone 空闲链表
                          │
              （稍后 mm_core_init → memblock_free_all()    [mm/memblock.c:2339]
               把 memblock 空闲页整体倒入 Buddy）
```

> [!NOTE] `bootmem_init()` 的挡位
> `bootmem_init()` 这个名称是历史遗留。它现在内部实际做的是 `sparse_init()`（构建 `mem_map` 数组）与 `zone_sizes_init()`（`free_area_init` 建立各 zone 的空闲链表）。**Buddy 真正拿到空闲页要等 `memblock_free_all()`** 把 memblock 空闲页整体倒入。

---

## 📐 3. 虚拟地址布局（VA 布局）

`PAGE_OFFSET / VMALLOC_START / FIXMAP` 的**数值定义**在 `arch/arm/include/asm/memory.h`，`mmu.c` 是**使用它们建映射**的人：

```
0x00000000 +---------------------------+
           |   用户空间 (TASK_SIZE)     |
           |   vgettimeofday/fault     |
0xC0000000 +---------------------------+
 PAGE_OFFSET（VMSPLIT_3G）
           |   lowmem direct map       | ← map_lowmem()
           |   VA = PA - PHYS_OFFSET   |
           |        + PAGE_OFFSET      |
           +---------------------------+ VMALLOC_START
           |   vmalloc / modules       | ← early_vmalloc() 可调大小 (mmu.c:1185)
           |   (VMALLOC_END 之下留      |
           |    VMALLOC_OFFSET 护栏)    |
           +---------------------------+
           |   PKMAP (highmem)         | ← kmap_init()
           |   FIXMAP                  | ← early_fixmap_init / shutdown
           |   vectors 0xffff0000      | ← devicemaps_init()
0xFFFFFFFF +---------------------------+
```

> [!IMPORTANT] IMX6ULL（512MB DDR @ `0x80000000`）实算
> `adjust_lowmem_bounds()` 用
> ```
> vmalloc_limit = VMALLOC_END - vmalloc_size - VMALLOC_OFFSET - PAGE_OFFSET + PHYS_OFFSET   (mmu.c:1223)
> ```
> 比较 memblock 各 range，得到 `arm_lowmem_limit = 0xA0000000`，即整个 512MB 全部进 lowmem，`high_memory = __va(0x9FFFFFFF) + 1`。

---

## 🔍 4. 各函数逐层拆解

### 4.1 `adjust_lowmem_bounds()`（mmu.c:1210）—— 不止"分低/高内存"

真实做 4 件事：

1. **首个 memblock range 起点若非 PMD 对齐** → `memblock_mark_nomap()`（无法用 section 直接映射的零头不映射）。
2. 算出 `arm_lowmem_limit`（≤ `vmalloc_limit` 的最大 PA 结束点）与 `high_memory`。
3. 算 `memblock_limit`：指向第一个非 PMD 对齐页，保证早期 `early_alloc()` 只从已映射的 PMD 内分配。
4. **若未开 HIGHMEM（或 VIPT aliasing cache）**：`memblock_remove()` 掉 lowmem_limit 之外的 RAM 并打印 `Ignoring RAM at ...` —— 这部分物理内存**直接作废**，而不是"交给 highmem 机制"。只有开了 HIGHMEM 才保留给上层。

> [!IMPORTANT] 正确的心智模型
> 不是"RAM 被切成 lowmem + highmem 两块映射"，而是"**lowmem 上限之外的内存要么留给（未映射的）highmem 页，要么被删除**"。

### 4.2 `prepare_page_table()`（mmu.c:1305）—— 清的是 PMD，不是"旧页表"

粒度为 PMD（2MB）整段清，分三段：

```c
for (addr = 0; addr < MODULES_VADDR; ...)
    pmd_clear();                        // 用户区全清
for ( ; addr < PAGE_OFFSET; ...)
    pmd_clear();                        // 内核 image 以下
for (addr = __phys_to_virt(end); addr < VMALLOC_START; ...)
    pmd_clear();                        // 第一个 memblock 之外的 lowmem
```

> [!WARNING] 关键细节：第一个 memory bank 不能清
> 注意：**第一个 memory bank 的直接映射不清**，因为代码此刻正跑在这块映射上（开了 MMU 的恒等映射由它替换沿用）。另外，XIP 时跳过 module 区的内核映射，KASAN 时跳过 shadow 区。

### 4.3 `map_lowmem()`（mmu.c:1502）—— direct map + 6 种 kernel 抠洞情形

对每个 memblock range 构造 `struct map_desc { pfn, virtual = __phys_to_virt(start), length, type = MT_MEMORY_RW }` → `create_mapping()`。

> [!CAUTION] 最易迷路但仍要抓住的主线：kernel 抠洞
> 真实代码里最重的一块注释（mmu.c:1518-1540）是 **kernel 镜像落在 VMALLOC 区时，要从 lowmem 映射中"凿掉" kernel 物理段**，理论上有 5~6 种位置关系（case 1~6，对应 kernel 与 memory range 的 start/end 交叠）。这是读这段代码时最该抓住的主线，别在这里迷失。

### 4.4 `map_kernel()`（mmu.c:1585）—— 只分两段，section 粒度

`.text RX / .rodata RO / .data RW / .bss RW+NX` 这种精细划分**不是这里做的**。真实代码只有两段：

| 区段 | 类型 | 内容 |
|---|---|---|
| `kernel_x_start..kernel_x_end` | `MT_MEMORY_RWX` | `.text + .init`（`round_up(__pa(__init_end))`） |
| `kernel_nx_start..kernel_nx_end` | `MT_MEMORY_RW` | 其余部分 |

且是 **section（1MB/2MB）粒度**。

> [!WARNING] 早期部分是"暂时可执行"，别误当永久权限
> 源注释（mmu.c:1602-1606）明确说明：section 向上取整导致**部分非可执行内存暂时被映射成可执行**，"只持续到稍后用 page 粒度重新映射整个内核为止"。真正的 RX/RO/RW 细分发生在 `paging_init` 之后（`patch_vivia_mappings` 等，以及 memblock → page 粒度重映射）。XIP 分支则把 flash 内容映射进 `MODULES_VADDR`（`MT_ROM`）。

### 4.5 `early_fixmap_shutdown()`（mmu.c:1747）—— 只"转正"，不"全拆"

```c
pte_offset_fixmap = pte_offset_late_fixmap;      // 切换到后期 fixmap 实现
pmd_clear(fixmap_pmd(va));                       // 只清掉早期 fixmap 用的那一个 PMD
for (i < __end_of_permanent_fixed_addresses) {
    pte = pte_offset_early_fixmap(...);
    if (pte_none || !L_PTE_MT_DEV_SHARED)
        continue;                                // 只认 ioremap 的设备映射
    map = { pfn, type = MT_DEVICE, length = PAGE_SIZE };
    create_mapping(&map);                        // 原地重建为正式映射
}
```

> [!NOTE] fixmap 机制继续存在
> 它把早期 fixmap 里属于**共享设备（`DEV_SHARED`，即 `early_ioremap` / debug console）**的页转成正式 `MT_DEVICE` 映射，然后拆除早期 fixmap 专用 PMD。**fixmap 机制本身继续存在**（后续 `fix_to_virt` 还用）。注意它并非"清掉所有早期 fixmap 再重建"。

### 4.6 `devicemaps_init()`（mmu.c:1391）—— vectors、DTB 与静态设备映射

- `early_alloc(2页)` 拿 vectors 页；
- `early_trap_init()` 清 `VMALLOC_START ~ fixmap 顶` 的 PMD（**保留早期 fixmap**；注释 mmu.c:1385-1389 明说此时若碰到设备映射会被清掉，因此只有 earlycon 能调试本函数）；
- **DTB**：`__atags_pointer` → `FDT_FIXED_BASE`（`MT_MEMORY_RO`）—— DT 物理地址即 uboot 传下来的 **fdt 指针**；
- `FLUSH_BASE / MINICACHE`（老 CPU 才编进来）；
- **vectors** → `0xffff0000`（`KUSER_HELPERS` 时 `MT_HIGH_VECTORS`）+ 第二页只读映射；
- `mdesc->map_io()` 或 `debug_ll_io_init()` ← 静态设备映射入口（IMX6 主要靠 DT + `ioremap`，此处仅 earlycon）；
- `fill_pmd_gaps()` PCI 空洞占位；
- `local_flush_tlb_all(); flush_cache_all()`；
- `early_abt_enable()` 打开异步异常。

> [!NOTE] DT 与 MMIO 的真实来源
> IMX6 的外设 MMIO **并不**在此处静态映射，而是靠 `of_iomap / ioremap` 运行时映射。这里静态映射的核心是 DT 只读节、vectors 与 earlycon。

### 4.7 `kmap_init()`（mmu.c:1491）—— 比字面窄

只有 8 行：HIGHMEM 时预分配 `PKMAP_BASE` 的 PTE 表 + 无论如何为 `FIXADDR_START` 预分配 PTE 表。它只是给 kmap / fixmap 准备**页表载体，不做任何映射**；真正的 PKMAP 填页在 `mm/highmem.c`。kmap ≠ vmalloc。

### 4.8 `early_paging_init()`（mmu.c:1654/1728）—— 只服务 LPAE 上的特殊平台

- `CONFIG_ARM_PV_FIXUP` 且 `CONFIG_ARM_LPAE`：调 `mdesc->pv_fixup()` 拿 offset，改 `kernel_sec_start/end`，然后关 MMU 跑一段汇编 `lpae_pgtables_remap_asm` **原地重建启动页表**，更新 `__pv_offset / __pv_phys_pfn_offset`。设计目标是 Keystone2 这类物理地址 > 4GB 的平台。
- 非 LPAE 分支（mmu.c:1728）：若平台仍返回非零 offset，直接打印"你的内核可能要崩了，祝你好运"并 `add_taint()`。

> [!WARNING] IMX6ULL 不涉及此路径
> 注意：IMX6ULL 不涉及此路径（`imx_v6_v7_defconfig` 未开 LPAE；DT 机器不提供 `pv_fixup`，offset 恒为 0 直接返回）。

### 4.9 `set_ptes()`（mmu.c:1824）—— 全架构 PTE 写入的统一出口

它不是 mmu.c 启动建映射专用的私有函数，而是**整个 ARM 内核（含用户态缺页）写 PTE 的统一入口**（`include/linux/pgtable.h` 的 weak 默认被本文件强符号覆盖）：

```c
if (addr < TASK_SIZE && pte_valid_user(pteval)) {
    if (!pte_special(pteval))
        __sync_icache_dcache(pteval);     // 用户可执行页 → 清/同步 I$
    ext |= PTE_EXT_NG;                    // 非全局，ASID 生效
}
set_pte_ext(ptep, pteval, ext);           // XPE 扩展位：AP/TEX 打包进 PMB，XN→PTE_EXT_XN
```

> [!NOTE] 它同时服务用户态映射
> 启动建映射的底层链条是：`create_mapping → __create_mapping (mmu.c:950) → alloc_init_p4d/pud/pmd（逐级，LPAE 才全走）→ alloc_init_pte (mmu.c:784) / __map_init_section (mmu.c:798) → set_ptes / set_pte_ext`。运行时缺页分配同样汇入这里 —— 把它定位成"页表写入硬件的最后一步"正确，但它同时服务于**用户态映射**。

### 4.10 `build_mem_type_table()`（mmu.c:463）—— 决定"映射长什么样"

`MT_DEVICE / MT_MEMORY_RW / MT_ROM …` 每个 `mem_type` 的 `.prot_pte / .prot_pmd` 在这里按 CPU 特性（`cache_viv/pipt`、`ecc`、`sam`、`usercachepolicy` 等命令行参数 `early_cachepolicy / nocache / nowb / ecc`，mmu.c:155-231）逐位修正。`create_mapping()` 的 `map.type` 最终就是查这张表得到硬件位。`pgprot_kernel / pgprot_user` 也从这里导出（mmu.c:69-70）。它是 `early_mm_init()` 的前半，也是所有映射硬件保护位的来源。

---

## 📊 5. 属性 / 粒度速查

| `map.type` | 粒度 | 典型用途 |
|---|---|---|
| `MT_MEMORY_RW` | section | lowmem direct map |
| `MT_MEMORY_RWX` | section | kernel `.text + .init`（早期，稍后 page 粒度细分） |
| `MT_ROM` | section | XIP flash |
| `MT_DEVICE` | page | MMIO（`early_fixmap_shutdown` 转正 / `map_io`） |
| `MT_HIGH_VECTORS` | page | `0xffff0000` |
| `MT_MEMORY_RO` | section | DTB（`FDT_FIXED_BASE`） |

---

## 🗺️ 6. 最终心智模型：启动路径总图

```
                 ARM Linux 启动（ VMSPLIT_3G, 非 LPAE）
                                    │
        head.S: swapper_pg_dir 恒等映射, 开 MMU（临时脚手架）
                                    │
           setup_arch ── early_fixmap_init ── early_mm_init
                                    │                    └ build_mem_type_table（属性表）
                     adjust_lowmem_bounds ×2 ── arm_memblock_init（页表进 memblock）
                                    │
                              paging_init
      ┌─────────────┬──────────────┼──────────────┬───────────────┐
 prepare_page   map_lowmem     map_kernel     fixmap 转正     devicemaps/kmap
      │             │              │              │               │
  清旧 PMD      direct map      section 级     DEV_SHARED       vectors/DTB/
 (保留首bank)  PA↔PAGE_OFFSET   RWX/RW         → 正式映射        earlycon MMIO
      └─────────────┴──────────────┴──────────────┴───────────────┘
                                    │  全部经 create_mapping → alloc_init_* → set_ptes
                             bootmem_init（sparse + zone 结构）
                                    │
                      memblock_free_all（空闲页倒入）
                                    ▼
                     mm/page_alloc.c — Buddy 开始接管物理页
```

> [!IMPORTANT] 边界一句话
> `mmu.c` 交出的是"一张可用的 `swapper_pg_dir` + 一段已被 memblock 记账的物理内存"；之后"哪些页空闲、order 分配、伙伴合并"才轮到 `mm/page_alloc.c`。**mmu.c 交出一张可用的内核页表(PA→VA)+ 一条 lowmem 上限线。**

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 梳理 mmu.c 施工链路前必过
> - [ ] **定位**：只搭 VA→PA 页面施工，页框分配（Buddy）在 `mm/page_alloc.c` 属后续
> - [ ] **`adjust_lowmem_bounds` 调用两次**：`arm_memblock_init` 前后各一次；节点不在 `paging_init` 内
> - [ ] **首 bank 保留**：`prepare_page_table` 不清第一个 memory bank（运行中替换沿用）
> - [ ] **低/high 心智模型**：是"lowmem 上限外内存删除或留 highmem"，不是"切成两块映射"
> - [ ] **map_kernel 两段**：仅 section 粒度，`MT_MEMORY_RWX`(.text+.init)/`MT_MEMORY_RW`；精细 RX/RO/RW 在之后做
> - [ ] **kernel 抠洞**：kernel 落 VMALLOC 区时从 lowmem 凿掉该段（case 1~6）
> - [ ] **fixmap 转正**：`early_fixmap_shutdown` 只转 `DEV_SHARED` 为 `MT_DEVICE`，fixmap 机制继续存在
> - [ ] **execmem 位置**：模块区在 `PAGE_OFFSET` 之下（非 vmalloc 层），见 [[arm mm init.c]]
> - [ ] **IMX6ULL**：未开 LPAE，`early_paging_init` 的 PV_FIXUP 路径不涉及
> - [ ] **Buddy 接管边界**：`memblock_free_all()` 后才轮到 `mm/page_alloc.c`

---

## 🔗 思考与延伸索引

- [[arm mm init.c]] — 同一链路的"收尾总管"：memblock/zone/RWX/init 回收 与 `arch_mm_preinit`
- [[页表预分配 mmu.c]] — 页表 PTE 缓存策略速查（uncached/buffered/writethrough/writeback/writealloc）
- [[Linux 内存管理]] — 通用 `mm/` 子系统（Buddy、vmalloc、page allocator）
- [[ARM 启动流程]] — head.S 恒等映射与开 MMU 的"临时脚手架"
- [[SWIOTLB]] — 设备 DMA 能力不足时的 bounce buffer 兜底

---

## 📚 参考资料

1. **Linux Kernel 源码** — `arch/arm/mm/mmu.c`（`create_mapping`、`paging_init`、各映射函数，含行号标注）.
2. **Linux Kernel 源码** — `arch/arm/include/asm/memory.h`（`PAGE_OFFSET`/`VMALLOC_START`/`FIXMAP` 数值定义）.
3. **Linux Kernel 源码** — `arch/arm/mm/init.c`（`bootmem_init`/`arm_memblock_init`，时序衔接）.
4. **Linux Kernel 源码** — `mm/init.c`、`mm/memblock.c:2339`（`memblock_free_all`、sparse/zone 建立）.
5. **NXP** — *i.MX 6ULL Reference Manual*（512MB DDR @ 0x80000000、`imx_v6_v7_defconfig` 内存配置）.