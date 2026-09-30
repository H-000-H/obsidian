---
title: 页表预分配 mmu.c 缓存策略速查
tags:
  - Kernel
  - ARM
  - MMU
  - Cache
  - Page-Table
status: In-progress
created: 2026-09-30
---

# 🧮 页表预分配 mmu.c 缓存策略速查

> [!INFO] 笔记概览
> - **定位**：页表预分配（`mmu.c`）中内存缓存策略的**一张速查表**，无冗长讲解、只供快速查用。用于在页表/物理内存映射时决定某段内存采用哪种缓存策略（uncached / buffered / writethrough / writeback / writealloc）。
> - **内容范围**：五种缓存策略在读命中、读不命中、写命中、写不命中时的行为，以及写回时机。
> - **推荐路径**：直接看下面 §1 速查表 → 对照 §2 关键差异点把握取舍 → 落地校验清单核对自己的策略选择。

---

## 💾 1. 缓存策略速查表

| 策略           | 读命中  | 读不命中  | 写命中        | 写不命中        | 写回时机         |
| ------------ | ---- | ----- | ---------- | ----------- | ------------ |
| uncached     | 直接访存 | 直接访存  | 直接访存       | 直接访存        | -            |
| buffered     | 直接访存 | 直接访存  | 写入写缓冲      | 写入写缓冲       | 写缓冲异步刷出      |
| writethrough | 走缓存  | 加载到缓存 | 缓存 + 内存同时写 | 直接写内存（不分配）  | 实时同步         |
| writeback    | 走缓存  | 加载到缓存 | 写缓存 + 标记脏位 | 直接写内存（不分配）  | 缓存行替换 / 主动刷新 |
| writealloc   | 走缓存  | 加载到缓存 | 写缓存 + 标记脏位 | 先加载到缓存，再写缓存 | 缓存行替换 / 主动刷新 |

---

## ⚖️ 2. 关键差异点

| 维度 | 说明 |
| :--- | :--- |
| **读写是否走缓存** | `uncached` / `buffered` 读不走缓存；`writethrough` / `writeback` / `writealloc` 读都走缓存 |
| **写不命中（写分配）** | `writethrough` 与 `writeback` 写不命中**直接写内存（不分配）**；只有 `writealloc` 会**先加载到缓存再写缓存** |
| **写回时机** | `buffered` 靠写缓冲异步刷出；`writethrough` 实时同步内存；`writeback`/`writealloc` 靠缓存行替换或主动刷新 |
| **脏位标记** | 仅 `writeback` / `writealloc` 写命中时"写缓存 + 标记脏位"，通过脏位延迟写回内存 |

> [!TIP] 快速记忆
> 从"最省电、无缓存"到"写回延时"可作如下排序：`uncached`（完全傍路）→ `buffered`（写缓冲）→ `writethrough`（每写同步）→ `writealloc`（写不命中先分配）→ `writeback`（延迟写回，性能最优但一致性责任最大）。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 选择缓存策略前必过
> - [ ] **用途匹配**：MMIO / 设备寄存器用 `uncached` 或 `buffered`；DMA 缓冲需核对一致性需求
> - [ ] **写分配语义**：确认写不命中时是否需要先分配到缓存（`writealloc`）还是直接写内存（`writethrough`/`writeback`）
> - [ ] **写回时机**：`writeback`/`writealloc` 必须安排好缓存刷新（cache flush / 行替换），避免脏数据滞留
> - [ ] **一致性**：`writethrough` 实时同步最稳但慢；`writeback` 高效但需要主动同步内存
> - [ ] **与页表映射对应**：在 `mmu.c` 页表/物理内存映射处按需选择对应 `mem_type`（见 [[arm mm init.c]] 的 `build_mem_type_table`）

---

## 🔗 思考与延伸索引

- [[mmu.c kernel起步]] — `map_lowmem`/`map_kernel` 映射时如何落地这些缓存属性
- [[arm mm init.c]] — `build_mem_type_table`（mmu.c:463）按 CPU/命令行为 `mem_type` 修正 `.prot_pte/.prot_pmd`，缓存策略在此编码为硬件位
- [[ARM Cache 一致性]] — DMA 与 CPU 视角下 `writeback`/`writealloc` 的 flush/clean 责任边界
- [[Linux 内存管理]] — cache 策略与页表属性（`W/WT/PAT` 等）映射到通用 MM

---

## 📚 参考资料

1. **ARM Architecture Reference Manual** — 缓存与写策略（write-through / write-back）、写缓冲语义.
2. **Linux Kernel 源码** — `arch/arm/mm/mmu.c`（`mem_type` 属性表与 `.prot_pte/.prot_pmd`）.