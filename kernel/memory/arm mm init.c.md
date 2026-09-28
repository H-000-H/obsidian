# ARM Linux `arch/arm/mm/init.c` 总结

> 文件：`arch/arm/mm/init.c`
> 
> 核心定位：**ARM 架构 MM（Memory Management）启动阶段的初始化与收尾总管。**
> 
> 它本身不是完整的 Linux MM 实现，而是负责把 ARM 平台的物理内存、Zone、memblock、内核权限、init 内存回收、execmem 等内容接到 Linux 通用 MM 上。

---

# 1. 整体职责

这个文件主要处理：

```text
Bootloader / DT
      │
      ▼
启动参数 / initrd
      │
      ▼
memblock
      │
      ▼
物理 RAM 范围确定
      │
      ▼
PFN / sparse memmap / struct page
      │
      ▼
ZONE_DMA / ZONE_NORMAL / ZONE_HIGHMEM
      │
      ▼
正式 page allocator
      │
      ├──────────────────────┐
      │                      │
      ▼                      ▼
内核 VA 权限收紧          init/initrd 回收
      │
      ▼
execmem 可执行内存区域配置
```

因此可以把它理解为：

> **ARM MM 启动阶段的“组织者”，而不是所有 MM 功能的实现者。**

Buddy、vmalloc、page allocator 等具体实现主要位于通用 `mm/` 子系统。

---

# 2. 启动参数与 initrd

## 2.1 `__clear_cr()`

```c
unsigned long __init __clear_cr(unsigned long mask)
{
    cr_alignment = cr_alignment & ~mask;
    return cr_alignment;
}
```

作用：

```text
修改 ARM 内核保存的软件 control register 状态
```

这里本身不是直接写 CP15 硬件寄存器。

真正的 CP15 control register 修改由其他 `set_cr()` 等机制完成。

---

# 3. initrd 启动参数解析

相关函数：

```c
parse_tag_initrd()
parse_tag_initrd2()
```

它们处理 ARM 老式 ATAG 启动参数：

```text
ATAG_INITRD
ATAG_INITRD2
       │
       ▼
phys_initrd_start
phys_initrd_size
```

`ATAG_INITRD` 还会进行虚拟地址 → 物理地址转换。

```text
Bootloader
   │
   └── initrd 地址 + 大小
             ↓
      Linux 记录 initrd
```

现代 ARM 更多使用 Device Tree `/chosen`，这里主要保留 ATAG 兼容支持。

---

# 4. 物理内存边界：`find_limits()`

```c
static void __init find_limits(unsigned long *min,
                               unsigned long *max_low,
                               unsigned long *max_high)
{
    *max_low = PFN_DOWN(memblock_get_current_limit());
    *min = PFN_UP(memblock_start_of_DRAM());
    *max_high = PFN_DOWN(memblock_end_of_DRAM());
}
```

得到三个重要 PFN：

```text
min
    ↓
DRAM 起始 PFN

max_low
    ↓
lowmem 结束 PFN

max_high
    ↓
整个 DRAM 结束 PFN
```

可以理解为：

```text
Physical RAM
0
│
├─────────────── lowmem ───────────────┤
│                                      │
min                                  max_low
│                                      │
├────────────── highmem ───────────────┤
                                       │
                                     max_high
```

注意：

```text
PFN = Physical Frame Number
PFN = Physical Address / PAGE_SIZE
```

---

# 5. DMA Zone

## 5.1 `setup_dma_zone()`

```c
if (mdesc->dma_zone_size) {
    arm_dma_zone_size = mdesc->dma_zone_size;
    arm_dma_limit = PHYS_OFFSET + arm_dma_zone_size - 1;
} else
    arm_dma_limit = 0xffffffff;

arm_dma_pfn_limit = arm_dma_limit >> PAGE_SHIFT;
```

核心：

```text
arm_dma_limit
    ↓
DMA 可以使用的最高物理地址

arm_dma_pfn_limit
    ↓
转换成 PFN
```

---

## 5.2 `ZONE_DMA`

在：

```c
zone_sizes_init()
```

中：

```c
max_zone_pfn[ZONE_DMA] =
    min(arm_dma_pfn_limit, max_low);
```

因此：

```text
ZONE_DMA
    ↓
满足 DMA 地址限制的物理 RAM

ZONE_NORMAL
    ↓
lowmem

ZONE_HIGHMEM
    ↓
超出永久 kernel direct map 的 RAM
```

重要：

> **DMA 是设备访问内存的能力，不是某种“DMA 专用内存”。**

`ZONE_DMA` 只是 Linux 给满足 DMA 地址约束的物理页面划出的 Zone。

---

# 6. `pfn_valid()`

```c
int pfn_valid(unsigned long pfn)
```

作用：

> 判断一个 PFN 是否对应实际存在的 RAM / 有效的 memory map。

核心逻辑：

```text
PFN
 │
 ▼
物理地址
 │
 ▼
检查是否落在 memblock.memory
 │
 ├── 是 → valid
 └── 否 → invalid
```

注意三个概念不要混：

```text
pfn_valid()
    ↓
这个物理页是否存在？

Zone
    ↓
这个物理页属于 DMA / NORMAL / HIGHMEM 哪个区域？

Kernel VA
    ↓
内核能不能通过永久 direct map 访问？
```

---

# 7. `arm_memblock_steal()`

```c
phys = memblock_phys_alloc(size, align);

memblock_phys_free(phys, size);

memblock_remove(phys, size);
```

它不是普通的：

```text
alloc → 使用 → free
```

而是：

```text
从 memblock 找一块物理内存
        ↓
从 memblock.memory 中永久移除
        ↓
以后普通内存管理不再管理这块
```

因此叫：

```text
steal
```

也就是：

> **把一段物理内存从 Linux 普通 RAM 管理范围中“偷走”。**

而且：

```c
arm_memblock_steal_permitted = false;
```

在 `arm_memblock_init()` 后会关闭这种操作。

---

# 8. I-Cache 检查

```c
check_cpu_icache_size()
```

读取 ARM CP15：

```text
CPU
 ↓
读取 Cache Type Register
 ↓
得到 I-Cache line size
 ↓
比较不同 CPU
```

如果多核 CPU 的 I-Cache line size 不一致，则启用对应 workaround。

这部分与物理内存分配关系不大，属于 ARM 特定的 cache 初始化兼容处理。

---

# 9. `arm_memblock_init()`

这是早期物理内存保留的重要函数。

```c
memblock_reserve(__pa(KERNEL_START),
                 KERNEL_END - KERNEL_START);

reserve_initrd_mem();

arm_mm_memblock_reserve();

mdesc->reserve();

early_init_fdt_scan_reserved_mem();

dma_contiguous_reserve(arm_dma_limit);
```

整体：

```text
                 memblock
                    │
       ┌────────────┼─────────────┐
       │            │             │
     Kernel        initrd      ARM MM
       │            │             │
       └────────────┼─────────────┘
                    │
              platform reserve
                    │
             DT reserved-memory
                    │
                  CMA
```

也就是说：

> **在正式 page allocator 接管 RAM 之前，先把绝对不能拿来分配的物理内存全部登记/保留。**

最后：

```c
arm_memblock_steal_permitted = false;
```

关闭 steal。

---

# 10. `bootmem_init()` —— 正式建立 page/zone 基础

```c
void __init bootmem_init(void)
{
    memblock_allow_resize();

    find_limits(...);

    early_memtest(...);

    sparse_init();

    zone_sizes_init(...);
}
```

这是整个文件非常重要的一个阶段。

---

## 10.1 `memblock_allow_resize()`

允许 memblock 自己扩展内部管理结构。

---

## 10.2 `find_limits()`

得到：

```text
min_low_pfn
max_low_pfn
max_pfn
```

---

## 10.3 `early_memtest()`

对低端物理内存进行早期测试。

---

## 10.4 `sparse_init()`

建立 sparse memory 对应的 `struct page` / memmap 基础。

也就是从：

```text
“这里有一段 RAM”
```

进一步变成：

```text
“Linux 有对应的 struct page 来描述这些物理页”
```

---

## 10.5 `zone_sizes_init()`

建立：

```text
ZONE_DMA
ZONE_NORMAL
ZONE_HIGHMEM
```

并最终：

```c
free_area_init(max_zone_pfn);
```

进入正式 page allocator 所需要的初始化阶段。

---

# 11. `bootmem_init()` 的核心意义

可以记成：

```text
memblock
  │
  │ 描述物理 RAM + reservations
  ▼
bootmem_init()
  │
  ├── sparse_init()
  │      ↓
  │   struct page / memmap
  │
  └── zone_sizes_init()
         ↓
      DMA/NORMAL/HIGHMEM
         ↓
      free_area_init()
```

所以：

> **`bootmem_init()` 是从“早期物理内存描述”走向“正式 Linux 页面管理结构”的关键桥梁。**

这里的函数名 `bootmem_init` 不意味着现代 Linux 仍然使用传统 bootmem allocator。

---

# 12. `arch_mm_preinit()`

```c
void __init arch_mm_preinit(void)
{
#ifdef CONFIG_ARM_LPAE
    swiotlb_init(max_pfn > arm_dma_pfn_limit,
                 SWIOTLB_VERBOSE);
#endif

    ...

    BUILD_BUG_ON(TASK_SIZE > MODULES_VADDR);
    BUG_ON(TASK_SIZE > MODULES_VADDR);

    ...
}
```

它的定位和 `bootmem_init()` 不一样。

```text
bootmem_init()
    ↓
建立正式 page/zone 内存管理基础

arch_mm_preinit()
    ↓
ARM 架构自己的 MM 前置准备和检查
```

---

# 13. SWIOTLB

核心：

```c
swiotlb_init(max_pfn > arm_dma_pfn_limit,
             SWIOTLB_VERBOSE);
```

判断：

```text
max_pfn > arm_dma_pfn_limit ?
```

即：

```text
整个 RAM 的最高地址
        >
DMA 能访问的最高地址？
```

如果是：

```text
系统 RAM
┌──────────────────────────────┬──────────────┐
│ DMA 可以访问                 │ DMA 访问不到  │
└──────────────────────────────┴──────────────┘
                               ↑
                         DMA address limit
```

就需要 SWIOTLB。

它提供 bounce buffer：

```text
CPU
 │
 │ copy
 ▼
低地址 bounce buffer
 │
 │ DMA
 ▼
设备
```

所以：

> **SWIOTLB 解决的是“设备 DMA 地址能力不足以直接访问某些 RAM”的问题。**

不是所有 HIGHMEM 都必须经过 SWIOTLB。

---

# 14. `arch_mm_preinit()` 的 VA 检查

检查：

```c
BUILD_BUG_ON(TASK_SIZE > MODULES_VADDR);
```

保证：

```text
用户地址空间结束
        ≤
MODULES_VADDR
```

也就是：

```text
0
│
├──────────── User ────────────┤
│                              │
TASK_SIZE
│
├──────────── Kernel ──────────┤
│
MODULES_VADDR
```

HIGHMEM 下还检查 PKMAP 区能否放得下：

```c
// asm/highmem.h:8
#define PKMAP_BASE  (PAGE_OFFSET - PMD_SIZE)

BUILD_BUG_ON(PKMAP_BASE + LAST_PKMAP * PAGE_SIZE > PAGE_OFFSET);
```

`PKMAP_BASE` 在 `PAGE_OFFSET` 之下（直接压在 lowmem direct map 起点上方）。这条断言真正保的是 **PKMAP 那一圈页表别从上方撞进 direct map 的起点 `PAGE_OFFSET`**，而不是“不越过 kernel/user 边界”。方向是：PKMAP 在 PAGE_OFFSET 之下向上生长，断言保证它不会顶到 PAGE_OFFSET。

---

# 15. Strict Kernel RWX

如果：

```c
CONFIG_STRICT_KERNEL_RWX
```

启用，就会进一步细化内核页表权限。

早期可能比较粗：

```text
Kernel mapping
        ↓
       RWX
```

后面变成：

```text
text       → RO + X
rodata     → RO + NX
data       → RW + NX
init       → RW + NX
```

---

# 16. `nx_perms[]`

主要做 NX：

```text
NX = No Execute
```

配置区域：

```text
PAGE_OFFSET → _stext
    NX

__init_begin → _sdata
    NX

rodata → __init_begin
    NX
```

也就是：

```text
区域                 权限
────────────────────────────
内核代码              X
内核代码之前的数据     NX
init 数据              NX
rodata                 NX
```

---

# 17. `ro_perms[]`

把：

```text
_stext → __init_begin
```

设置为只读。

这个区域主要包含：

```text
kernel text
rodata
```

最终目标：

```text
.text
    RO + X

.rodata
    RO + NX
```

---

# 18. `section_update()`

这是实际修改 PMD 权限的底层函数。

核心：

```c
pmd = ...
pmd[...] =
    __pmd((old & mask) | prot);
```

然后：

```c
flush_pmd_entry(pmd);
local_flush_tlb_kernel_range(...);
```

流程：

```text
找到 kernel VA 对应 PMD
        ↓
修改权限位
        ↓
写回 PMD
        ↓
刷新相关 cache/TLB
```

所以：

> **这里真正开始修改页表中的权限属性。**

---

# 19. `arch_has_strict_perms()`

检查当前 ARM CPU 是否支持严格权限：

```c
cpu_architecture() >= ARMv6
```

并检查：

```c
CR_XP
```

如果不满足，就不执行严格权限设置。

---

# 20. `set_section_perms()`

它负责遍历权限描述：

```text
section_perm[]
     │
     ▼
检查 section 对齐
     │
     ▼
一个 section 一个 section
     │
     ▼
section_update()
```

因此：

```text
set_section_perms()
        ↓
    遍历区域
        ↓
 section_update()
        ↓
     修改 PMD
```

---

# 21. `update_sections_early()`

这里有一个重要点：

> kernel mapping 会存在于各个进程的页表中。

所以不能只修改：

```text
init_mm
```

而是要更新各个进程对应的 kernel section。

代码会遍历：

```c
for_each_process()
```

并对有 `mm` 的线程执行：

```c
set_section_perms(...)
```

同时更新：

```text
current->active_mm
init_mm
```

---

# 22. `stop_machine()`

权限修改通过：

```c
stop_machine(...)
```

执行。

原因是：

```text
CPU0
 └── 修改 kernel page table

CPU1 ──暂停
CPU2 ──暂停
CPU3 ──暂停
```

避免其他 CPU 同时使用正在修改的内核映射。

---

# 23. `fix_kernmem_perms()`

```c
update_sections_early(nx_perms, ...);
```

主要作用：

```text
把不应该执行的 kernel memory 设置为 NX
```

即：

```text
RWX
 ↓
RW + NX
```

---

# 24. `mark_rodata_ro()`

```c
update_sections_early(ro_perms, ...);
```

主要作用：

```text
kernel text / rodata
        ↓
设置 RO
```

然后：

```c
arm_debug_checkwx();
```

检查是否存在不应该出现的可写可执行区域。

---

# 25. `free_initmem()`

```c
void free_initmem(void)
{
    fix_kernmem_perms();

    poison_init_mem(...);

    free_initmem_default(-1);
}
```

这是启动完成后的重要收尾。

顺序：

```text
__init memory
     │
     ▼
fix_kernmem_perms()
     │
     ▼
权限收紧
     │
     ▼
poison_init_mem()
     │
     ▼
填充非法指令
     │
     ▼
free_initmem_default()
     │
     ▼
归还 Buddy / page allocator
```

---

# 26. 为什么要 poison？

```c
*p++ = 0xe7fddef0;
```

把 init memory 填成 undefined instruction。

目的：

> 如果后面错误地跳回已经释放的 `__init` 代码，尽可能快速暴露错误，而不是悄悄执行旧代码。

关键顺序：

```text
poison
  ↓
free
```

不能反过来。

因为：

```text
free
 ↓
Buddy 接管
 ↓
页面可能马上分配给别人
 ↓
再去 poison
 ↓
可能破坏别人的数据
```

---

# 27. `free_initrd_mem()`

initrd 回收：

```c
poison_init_mem(...);

free_reserved_area(..., "initrd");
```

流程：

```text
initrd
  ↓
对齐 PAGE
  ↓
poison
  ↓
free_reserved_area()
  ↓
重新成为可分配 RAM
```

---

# 28. `free_initmem()` 与 `free_initrd_mem()` 的区别

||`free_initmem()`|`free_initrd_mem()`|
|---|---|---|
|对象|kernel `__init` 区域|initrd|
|权限收紧|有|无需同样处理|
|poison|有|有|
|最终释放|`free_initmem_default()`|`free_reserved_area()`|
|本质|回收启动阶段 kernel init 代码/数据|回收临时 initrd 数据|

核心区别：

```text
__init
    ↓
内核自己的启动代码/数据

initrd
    ↓
Bootloader 提供的临时文件系统镜像
```

---

# 29. poison 后页面还能不能使用？

能。

`poison` 只是：

```text
把旧内容覆盖成特殊值
```

并不是：

```text
永久标记这个物理页不能使用
```

之后：

```text
Buddy
 ↓
重新分配
 ↓
可以再次使用
```

但：

```c
alloc_page(GFP_KERNEL)
```

**不保证得到清零页面**。

因此可能看到：

```text
旧内容
poison 值
其他旧数据
```

如果需要零初始化：

```c
alloc_page(GFP_KERNEL | __GFP_ZERO)
```

或者自己初始化后再使用。

---

# 30. execmem 区域：位于 PAGE_OFFSET 之下、贴近 kernel text

最后是：

```c
execmem_arch_setup()
```

作用不是建立整个 kernel VA layout，而是：

> **告诉通用 execmem 子系统：ARM 上可执行内存应该使用哪一段 kernel VA。**

默认：

```c
.start = MODULES_VADDR
.end   = MODULES_END
.pgprot = PAGE_KERNEL_EXEC
```

这两个宏定义在 `arch/arm/include/asm/memory.h:60`：

```c
#define MODULES_VADDR  (PAGE_OFFSET - SZ_16M)     // Thumb2 为 PAGE_OFFSET - SZ_8M
#define MODULES_END    (PAGE_OFFSET - PMD_SIZE)   // 非 HIGHMEM 时为 PAGE_OFFSET
```

**execmem / MODULES 区并不在 vmalloc 那一层，而是在 `PAGE_OFFSET` 之下**，紧贴在用户空间顶部、direct map 起点的正下方：

```text
TASK_SIZE (0xBFFFFFFF)
    ┌────────────────────────┐
    │    用户空间（最高处）     │
0xBF000000 ── MODULES_VADDR ──┐
    │  MODULES / execmem     │  ← PAGE_KERNEL_EXEC，bpf、module_alloc 在这
0xBFE00000 ── MODULES_END ────┘  （HIGHMEM 时；PMD_SIZE 之下的 PKMAP 共用顶端 1MB）
    │  PKMAP 区（HIGHMEM）    │  ← PKMAP_BASE = PAGE_OFFSET - PMD_SIZE
0xC0000000 ── PAGE_OFFSET ────┐
    │  lowmem direct map     │
    └────────────────────────┘
```

为什么要贴着 kernel text 放：ARM 32 位 `b/bl` 指令寻址只有 **±32MB**，把模块/可执行区放在内核 text 16MB 之侧，模块**不用 PLT 就能直接 `bl` 到内核符号**（`bl` 偏移可达 ±32MB）。这是 execmem 区域选址的根本原因。

---

# 31. `CONFIG_ARM_MODULE_PLTS`

`fallback` 到 vmalloc 区，是 **`CONFIG_ARM_MODULE_PLTS` 打开后**才发生的事（`init.c:475-478`）：

```c
fallback_start = VMALLOC_START;
fallback_end   = VMALLOC_END;
```

也就是说：

```text
默认：      MODULES_VADDR ~ MODULES_END   ← 主用于普通模块
fallback：  VMALLOC_START ~ VMALLOC_END   ← 仅 CONFIG_ARM_MODULE_PLTS 时，
                                            模块足够近可直接 bl 内核符号；
                                            否则装不下时回退到 vmalloc 并打 PLT
```

两者是**“默认模块区”与“开启 PLTS 后的回退区”**的关系，不是平级的两块；且默认模块区在 PAGE_OFFSET 之下，不是 vmalloc 那一层。

---

# 32. XIP Kernel

如果：

```c
CONFIG_XIP_KERNEL
```

则：

```c
#define MODULES_VADDR \
    (((unsigned long)_exiprom + ~PMD_MASK) & PMD_MASK)
```

也就是重新定义 `MODULES_VADDR`。

源码注释说明：

> XIP kernel text 映射到 module area，使 modules 等代码可以不依赖间接 relocation。

因此 XIP 下：

```text
Kernel text
    ↓
ROM / Flash
    ↓
映射到特定 kernel VA
```

这里主要是 **VA 布局兼容问题**，不是简单的“模块全部放 ROM”。

---

# 33. 整个 `init.c` 在真实时序中的位置

`init.c` 的各阶段散落在 `start_kernel / mm_core_init / kernel_init` 三个入口里，真实时序如下：

```text
start_kernel
 ├─ setup_arch [arch/arm/kernel/setup.c]
 │    ├─ early_mm_init / adjust_lowmem_bounds ×2 / arm_memblock_init   init.c:187
 │    ├─ paging_init ──► bootmem_init   init.c:209   ←★ bootmem_init 唯一调用点 mmu.c:1812
 │    │                        ├─ find_limits → min_low_pfn / max_low_pfn / max_pfn
 │    │                        ├─ early_memtest  (memtest= 参数控制，默认关)
 │    │                        ├─ sparse_init    → struct page 数组
 │    │                        └─ zone_sizes_init → free_area_init（只 ZONE_NORMAL…）
 │    └─ ...
 │
 ├─ mm_core_init [mm/mm_init.c:2679]
 │    ├─ arch_mm_preinit()    init.c:243   ← SA1111 释放 / VA 边界断言（仅 LPAE 才动 swiotlb）
 │    ├─ build_all_zonelists
 │    ├─ memblock_free_all()               ←★ Buddy 此刻才真正收到空闲页
 │    ├─ mem_init / kmem_cache_init / vmalloc_init
 │    └─ execmem_init() → execmem_arch_setup()   init.c:471
 │
 └─ kernel_init [init/main.c:1474]   ← 进入用户态之前
      ├─ free_initmem()        init.c:433
      │    ├─ fix_kernmem_perms() → stop_machine → nx_perms → PMD 打 XN
      │    ├─ poison_init_mem(0xe7fddef0)
      │    └─ free_initmem_default(-1)   → 归还 Buddy
      └─ mark_readonly() [main.c:1447]
           └─ mark_rodata_ro() init.c:423 → stop_machine → ro_perms → PMD 打 RO
```

> [!important] 关键顺序
> - `free_initmem()` **先**、`mark_rodata_ro()` **后**（`main.c:1492/1493`），两者同处 `kernel_init()`。
> - **Buddy 在 `mm_core_init` 里的 `memblock_free_all()` 就已正式开工**；RWX 收紧（`fix_kernmem_perms / mark_rodata_ro`）发生在内核已跑起来、initcall 都执行完之后的 `kernel_init()` 阶段。不要把 RWX 收紧画在 Buddy 之前。

---

# 34. 最重要的几个“分层”

学习这个文件时，建议牢牢记住下面这几个区别。

## ① memblock

```text
早期物理内存管理
```

负责：

```text
RAM 有哪些
哪些被 kernel 占用
哪些是 initrd
哪些是 reserved
哪些给 CMA
```

---

## ② sparse memmap / `struct page`

```text
建立 Linux 对物理页的描述
```

即：

```text
Physical Page
      ↕
struct page
```

---

## ③ Zone

```text
物理页面分类
```

主要：

```text
ZONE_DMA
ZONE_NORMAL
ZONE_HIGHMEM
```

---

## ④ Buddy / page allocator

```text
正式物理页分配器
```

`init.c` 负责把它初始化起来，但具体 allocator 实现不在这个文件里。

---

## ⑤ MMU / page table

负责：

```text
VA → PA
```

以及：

```text
RWX / RO / NX
```

---

## ⑥ SWIOTLB

解决：

```text
设备 DMA 地址能力
        ↓
无法直接访问某些物理 RAM
```

---

## ⑦ execmem

负责：

```text
kernel executable virtual memory
```

例如：

```text
MODULES_VADDR ~ MODULES_END
```

---

# 35. 一句话记忆整个文件

```text
init.c
=
“ARM 启动时把物理 RAM 从 memblock
逐步接入正式 Linux MM，
同时把 kernel VA 权限收紧，
最后释放 init/initrd 并配置 execmem。”
```

或者更短：

```text
发现/保留 RAM
      ↓
建立 struct page
      ↓
建立 Zone
      ↓
进入正式 page 管理
      ↓
处理 DMA
      ↓
收紧 Kernel RWX
      ↓
释放 __init / initrd
      ↓
配置 executable memory
```

> 这就是这个 `arch/arm/mm/init.c` 最值得掌握的主线。源码中的具体函数分别对应上述阶段。