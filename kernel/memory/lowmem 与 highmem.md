---
tags:
  - linux-kernel
  - arm
  - memory
  - highmem
  - lowmem
  - imx6ull
---

# lowmem 与 highmem：32 位内核的地址分界

> [!tip] 一句话理解
> **low 是永久地区，high 是针对 VA 不够用的临时解决方案。**
>
> - **lowmem（永久）**：给了一块常驻的虚拟地址（direct map），`PA ↔ VA` 随时可线性换算、可直接用指针解引用。
> - **highmem（临时）**：内核 VA 空间塞不下这些 RAM，因数没有常驻地址，要用时只能“临时借一张 PTE”映射，用完立刻还。
>
> 一句话：**low = 常驻直映射的永久地界；high = 现借现还、专治 VA 不够的临时暂存区。**

## 用一张图看懂 low / high 的分工

```
       内核 VA 空间（仅 1GB）
0xC0000000 +---------------------------+
           |     lowmem（永久）        | ← 常驻 direct map
           |   直接映射全部"装得下"的RAM |   __va()/__pa() 随时可算
           +---------------------------+ ─── 分界线 arm_lowmem_limit
           |     highmem（临时）        | ← 只留 struct page，
           |   不常驻映射，想用先借     |   用 kmap 临时映射，用完即还
           +---------------------------+
```

这个划分做到了两件事：

1. **保证关键路径永远有地址可用**：内核自己跑、`kmalloc`、`__va/__pa`、页表维护都只跟 lowmem 打交道，稳定、直接、无后顾之忧。
2. **把"装不下的余量"外包给高地址的临时机制**：页缓存、用户页、DMA——这些既不用内核直接解引用、又能等访问时才临时借映射的页，全塞进 highmem，不占 1GB VA 的常驻名额。

## 为什么会有这条线

32 位 ARM 内核的 VA 空间只有 **1GB**（3G/1G 分割，`PAGE_OFFSET = 0xC0000000`）。这 1GB 要塞下 `direct map + vmalloc + modules + PKMAP + FIXMAP + vectors`，装不下所有 RAM。于是内核画一条线：

- **线以下 = lowmem**：常驻 direct map，随时可直接寻址。
- **线以上 = highmem**：内核 VA 空间里**没有它的格子**，要用只能“临时借一张 PTE”。

> 没有给 highmem 一块常驻的虚拟地址，是它一切麻烦的根源。

## 这条线在代码里是谁

| 变量 | 谁算的 | 含义 |
|---|---|---|
| `arm_lowmem_limit` | `adjust_lowmem_bounds()` `mmu.c:1276` | **PA 侧**的分界线 |
| `high_memory` | `mmu.c:1278` `__va(arm_lowmem_limit-1)+1` | direct map 的顶，**仅此一个用途** |
| `max_low_pfn / max_pfn` | `find_limits() → zone_sizes_init()` `mm/init.c:111` | zone 划分：`ZONE_NORMAL` vs `ZONE_HIGHMEM` |

`zone_sizes_init()` 里看得很清楚（`mm/init.c:118-121`）：

```c
max_zone_pfn[ZONE_NORMAL]  = max_low;    // lowmem 的页
max_zone_pfn[ZONE_HIGHMEM] = max_high;   // 线以上的页
```

## 实际差别逐项对照

| | **lowmem** | **highmem** |
|---|---|---|
| `__va() / phys_to_virt()` | ✅ 线性可算，`memory.h:265` | ❌ 算出来的 VA 越界，无映射 |
| `virt_to_phys() / __pa()` | ✅ `memory.h:241` | ❌ 反过来 `page_address()` 对它返回 `NULL` |
| `virt_to_page(kaddr)` | ✅ | ❌（`memory.h:364`，只能 `pfn_to_page`） |
| `kmalloc()` | ✅ 天然满足 | ❌ 内部要 direct map 地址，默认不从高于 `high_memory` 分配 |
| `alloc_page()` | ✅ | ✅（拿到 `struct page`，但拿不到内核指针） |
| `kmap() / kmap_local()` | 不需要 | 唯一访问手段，走 PKMAP/FIXMAP 槽，用完必须 `kunmap` |
| `vm_insert_page()` | ✅ | ✅（vmalloc 区自己填 PTE，不依赖 direct map） |
| 老式页表里存 PA | 可以 | 绝不能把它的 `page_address()` 写进页表 |

> [!important] 核心机制
> `pfn_to_page()` 在 sparsemem 下是**查表**（vmemmap / section 数组），low/high 都能拿到 `struct page`；真正的分水岭是 `virt_to_pfn()` 这种**纯算术转换**（`memory.h:290`）只对 lowmem 成立。
>
> 所以内核里凡“靠加减法翻译地址”的地方，都必须先问一句：**这页在 `high_memory` 以下吗？**

## highmem 存在的代价

- 驱动拿到一个 highmem 页想做 DMA，必须先 `kmap_atomic()` 借地址，且该槽位不能睡眠、不能嵌套过深。
- `GFP_KERNEL` 分配普通内存时，Buddy 会**优先考虑低 zone**，避免无谓地用难访问的高页。
- 早期启动阶段完全不能用 highmem：`memblock_set_current_limit()`（`mmu.c:1794/1302`）把 memblock 分配上限压到 lowmem 内，就是为了让 `early_alloc()` 只会挑“已映射好的物理页”。

highmem 页主要在两类场景被使用，而这两类都**不需要内核 VA**：

### ① 用户态 / 页缓存路径（占绝大多数）

```
read() → alloc_page() 拿到 highmem 的 struct page
       → 直接把 PFN 填进【用户进程】页表的 PTE
       → 用户访问自己的 VA，MMU 走的是用户页表，跟内核 direct map 无关
```

页缓存在 highmem 里堆几百 MB 完全正常，内核从头到尾只握着 `struct page *`，从不解引用它的 `page_address()`。

### ② 内核真要碰内容 → 才临时映射

```c
void *k = kmap_local_page(page);   // 借一个 PKMAP/FIXMAP 槽，填 PTE
memcpy(...);                        // 现在有内核指针可用
kunmap_local(k);                    // 立刻还槽
```

约束就是从这里来的：**`kmap_atomic()` 用 FIXMAP 槽，不可睡眠；`kmap()` 用 PKMAP 循环槽，可睡眠但槽有限、有 contention**。所以代码里到处是 `if (PageHighMem(page)) ...` 的分支——这不是“要不要用这页”，而是“要不要为它借一次地址”。

### ③ DMA：第三种情况，很多根本不用映射

```
dma_map_page() 只算出 bus address(= PA) 交给设备
设备用自己的地址线访问 PA，不看内核 VA
```

只有 32 位 DMA mask 够不着的 PA，才需要 **swiotlb**：把数据先 `memcpy` 到 lowmem 的 **bounce buffer**。这就是 `swiotlb_init(max_pfn > arm_dma_pfn_limit, ...)`（`arch/arm/mm/init.c:244-246`）干的事——**RAM 超出设备可达范围才启用**。

## 三句话收尾

| | 谁在访问 | 需要内核 VA 吗 |
|---|---|---|
| 页缓存 / 用户匿名页 | 用户进程页表 | ❌ |
| 内核逻辑要读内容 | `page_address()` | ✅ 必须 `kmap` |
| 外设 DMA | 设备地址线 | ❌（除非过 swiotlb） |

> lowmem 的本质优势不是“能被分配到”，而是**“随时可以用指针解引用”**。highmem 是被剥夺了这个特权，换回 32 位内核 VA 空间不够用的余量。