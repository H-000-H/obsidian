---
title: lowmem 与 highmem：32 位内核的地址分界
tags:
  - linux-kernel
  - arm
  - memory
  - mmu
  - highmem
  - lowmem
  - imx6ull
  - kmap
  - swiotlb
status: Completed
kernel: "7.3.0-rc6 (mainline)"
---

# 💾 lowmem 与 highmem：32 位内核的地址分界

> [!INFO] 笔记概览
> - **定位**：32 位 ARM 内核把 1GB 内核 VA 空间划成 lowmem / highmem 两带，并立下 `arm_lowmem_limit` 分界线。核心一句话——**low 是永久地区，high 是针对 VA 不够用的临时解决方案。**
> - **内容范围**：low/high 分工全景图 · 分界线成因（3G/1G 分割）· 代码里的分界变量与函数 · 逐项能力对照表 · highmem 的存在代价 · 三大访问场景（页缓存 / 内核临时映射 / DMA）
> - **推荐路径**：§1 分工全景 → §2 分界成因 → §3 代码实现 → §4 能力对照 → §5 highmem 的代价 → §6 三句话收尾 → §7 校验与延伸

> [!IMPORTANT] 核心一句话总结
> - **lowmem（永久）**：给了一块常驻的虚拟地址（direct map），`PA ↔ VA` 随时可线性换算、可直接用指针解引用。
> - **highmem（临时）**：内核 VA 空间塞不下这些 RAM，因数没有常驻地址，要用时只能"临时借一张 PTE"映射，用完立刻还。
> - 一句话：**low = 常驻直映射的永久地界；high = 现借现还、专治 VA 不够的临时暂存区。**

---

## 🗺️ 1. 用一张图看懂 low / high 的分工

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

---

## ❓ 2. 为什么会有这条线

32 位 ARM 内核的 VA 空间只有 **1GB**（3G/1G 分割，`PAGE_OFFSET = 0xC0000000`）。这 1GB 要塞下 `direct map + vmalloc + modules + PKMAP + FIXMAP + vectors`，装不下所有 RAM。于是内核画一条线：

- **线以下 = lowmem**：常驻 direct map，随时可直接寻址。
- **线以上 = highmem**：内核 VA 空间里**没有它的格子**，要用只能"临时借一张 PTE"。

> [!NOTE] 麻烦的根源
> 没有给 highmem 一块常驻的虚拟地址，是它一切麻烦的根源。理解 low/high，本质就是理解「这块 VA 为什么给不了、又凭什么临时借」。

---

## 🔍 3. 这条线在代码里是谁

| 变量 | 谁算的 | 含义 |
|---|---|---|
| `arm_lowmem_limit` | `adjust_lowmem_bounds()` `mmu.c:1244` | **PA 侧**的分界线 |
| `high_memory` | `mmu.c:1246` `__va(arm_lowmem_limit-1)+1` | direct map 的顶，**仅此一个用途** |
| `max_low_pfn / max_pfn` | `find_limits() → arch_zone_limits_init()` `mm/init.c:110` | zone 划分：`ZONE_NORMAL` vs `ZONE_HIGHMEM` |

`arch_zone_limits_init()` 里看得很清楚（`mm/init.c:113-117`）：

```c
max_zone_pfn[ZONE_NORMAL]  = max_low;    // lowmem 的页
max_zone_pfn[ZONE_HIGHMEM] = max_high;   // 线以上的页
```

> [!TIP] 阅读提示
> 分界线本身在**物理地址（PA）侧**由 `arm_lowmem_limit` 度量，而 `high_memory` 只是它的一个 VA 换算副产物，用途固定为「direct map 顶」，不应把它当作通用边界到处依赖。

---

## ⚖️ 4. 实际差别逐项对照

| | **lowmem** | **highmem** |
|---|---|---|
| `__va() / phys_to_virt()` | ✅ 线性可算，`memory.h:319/329` | ❌ 算出来的 VA 越界，无映射 |
| `virt_to_phys() / __pa()` | ✅ `memory.h:313/327` | ❌ 反过来 `page_address()` 对它返回 `NULL` |
| `virt_to_page(kaddr)` | ✅ | ❌（`memory.h:384`，只能 `pfn_to_page`） |
| `kmalloc()` | ✅ 天然满足 | ❌ 内部要 direct map 地址，默认不从高于 `high_memory` 分配 |
| `alloc_page()` | ✅ | ✅（拿到 `struct page`，但拿不到内核指针） |
| `kmap() / kmap_local()` | 不需要 | 唯一访问手段，走 PKMAP/FIXMAP 槽，用完必须 `kunmap` |
| `vm_insert_page()` | ✅ | ✅（vmalloc 区自己填 PTE，不依赖 direct map） |
| 老式页表里存 PA | 可以 | 绝不能把它的 `page_address()` 写进页表 |

> [!IMPORTANT] 核心机制
> `pfn_to_page()` 在 sparsemem 下是**查表**（vmemmap / section 数组），low/high 都能拿到 `struct page`；真正的分水岭是 `virt_to_pfn()` 这种**纯算术转换**（`memory.h:290`）只对 lowmem 成立。
>
> 所以内核里凡"靠加减法翻译地址"的地方，都必须先问一句：**这页在 `high_memory` 以下吗？**

> [!CAUTION] 警示：别拿 highmem 走线性换算
> 线性换算 API（`__va` / `phys_to_virt` / `__pa` / `virt_to_phys`）是给 lowmem 准备的算术捷径。对 highmem 页：
> - `phys_to_virt` 算出的 VA 越界、无映射；`virt_to_phys` 与 `page_address()` 返回 `NULL`，一解引用就崩。
> - 老式页表若把 highmem 的 `page_address()` 当 PA 写进去，等于把「临时 VA」当「物理地址」存，彻底错乱。
>
> 处理 highmem 页的唯一正路是 `pfn_to_page()` 拿到 `struct page`，再用 `kmap()` 系借临时 VA——**不靠加减法翻译，靠查表 + 借射**。

---

## 💸 5. highmem 存在的代价

- 驱动拿到一个 highmem 页想做 DMA，必须先 `kmap_atomic()` 借地址，且该槽位不能睡眠、不能嵌套过深。
- `GFP_KERNEL` 分配普通内存时，Buddy 会**优先考虑低 zone**，避免无谓地用难访问的高页。
- 早期启动阶段完全不能用 highmem：`memblock_set_current_limit()`（`mmu.c:1760/1270`）把 memblock 分配上限压到 lowmem 内，就是为了让 `early_alloc()` 只会挑"已映射好的物理页"。

> [!WARNING] kmap 槽位的纪律
> `kmap_atomic()` 走 FIXMAP 槽，槽在中断/原子上下文里**不可睡眠、不可长时间占用**；用完必须立刻 `kunmap_local()` 归还，不可嵌套过深。若在该槽位上睡眠或嵌套过深，会破坏映射一致性甚至 BUG。

highmem 页主要在两类场景被使用，而这两类都**不需要内核 VA**：

### 5.1 用户态 / 页缓存路径（占绝大多数）

```
read() → alloc_page() 拿到 highmem 的 struct page
       → 直接把 PFN 填进【用户进程】页表的 PTE
       → 用户访问自己的 VA，MMU 走的是用户页表，跟内核 direct map 无关
```

页缓存在 highmem 里堆几百 MB 完全正常，内核从头到尾只握着 `struct page *`，从不解引用它的 `page_address()`。

### 5.2 内核真要碰内容 → 才临时映射

```c
void *k = kmap_local_page(page);   // 借一个 PKMAP/FIXMAP 槽，填 PTE
memcpy(...);                        // 现在有内核指针可用
kunmap_local(k);                    // 立刻还槽
```

约束就是从这里来的：**`kmap_atomic()` 用 FIXMAP 槽，不可睡眠；`kmap()` 用 PKMAP 循环槽，可睡眠但槽有限、有 contention**。所以代码里到处是 `if (PageHighMem(page)) ...` 的分支——这不是"要不要用这页"，而是"要不要为它借一次地址"。

### 5.3 DMA：第三种情况，很多根本不用映射

```
dma_map_page() 只算出 bus address(= PA) 交给设备
设备用自己的地址线访问 PA，不看内核 VA
```

> [!NOTE] swiotlb 只在够不着时才介入
> 只有 32 位 DMA mask 够不着的 PA，才需要 **swiotlb**：把数据先 `memcpy` 到 lowmem 的 **bounce buffer**。这就是 `swiotlb_init(max_pfn > arm_dma_pfn_limit, ...)`（`arch/arm/mm/init.c:226`）干的事——**RAM 超出设备可达范围才启用**。

---

## 📌 6. 三句话收尾

| | 谁在访问 | 需要内核 VA 吗 |
|---|---|---|
| 页缓存 / 用户匿名页 | 用户进程页表 | ❌ |
| 内核逻辑要读内容 | `page_address()` | ✅ 必须 `kmap` |
| 外设 DMA | 设备地址线 | ❌（除非过 swiotlb） |

> [!SUCCESS] 一句话本质
> lowmem 的本质优势不是"能被分配到"，而是**"随时可以用指针解引用"**。highmem 是被剥夺了这个特权，换回 32 位内核 VA 空间不够用的余量。

---

## 🔗 思考与延伸索引

- [[KERNEL_SET_UP]] — 内核启动进入 C 世界前的页表与内存准备
- [[arm mm init.c]] — `zone_sizes_init`、`adjust_lowmem_bounds` 的进一步展开
- [[mmu.c kernel起步]] — `direct map + vmalloc + PKMAP + FIXMAP + vectors` 布局与分界线的由来
- [[kernel mm视图]] — 1GB 虚拟地址空间全景图与 lowmem 生死线的确立过程
- [[DMA、Buffer 与流量控制]] — DMA 映射、bounce buffer 与 swiotlb 的关联

---

## 📚 参考资料

1. **Linux Kernel 源码** — `arch/arm/mm/mmu.c`（`adjust_lowmem_bounds`、`memblock_set_current_limit`）.
2. **Linux Kernel 源码** — `arch/arm/mm/init.c`（`find_limits`、`zone_sizes_init`、`swiotlb_init`）.
3. **Linux Kernel 源码** — `arch/arm/include/asm/memory.h`（`__va`、`__pa`、`virt_to_page`、`virt_to_pfn`、`page_address`）.
4. **Linux Kernel 源码** — `mm/highmem.c`（`kmap` / `kmap_atomic` / PKMAP / FIXMAP 槽），`mm/page_alloc.c`（Buddy 低 zone 优先）。
5. **Documentation/arm/memory.rst** — ARM 32 位虚拟内存布局官方说明.
