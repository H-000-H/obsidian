---
title: ARM Kernel head.S 内核启动 学习笔记
tags:
  - Kernel
  - ARM
  - Boot
  - Assembly
  - MMU
  - Memory
  - SMP
status: Completed
created: 2026-09-30
kernel: "7.3.0-rc6 (mainline)"
---

# 🏗️ ARM Kernel head.S 内核启动 学习笔记

> [!INFO] 笔记概览
> - **定位**：ARM32 内核本体 `arch/arm/kernel/head.S` 与 `__create_page_tables` 的逐阶段剖析——从「zImage 自解压临时开 MMU」到 `stext` 无栈裸机手搭页表、经 1:1 恒等映射跃迁高位虚拟地址、最终进入 C 语言虚拟世界。
> - **内容范围**：两幕戏对照 · 自解压阶段 MMU/页表 · `stext` 裸机期（入参/寄存器 · SMP_ON_UP 热补丁 · pv_table）· `__create_page_tables` · `__enable_mmu`/`__turn_mmu_on` · 栈确立（主核/从核 idle 任务栈）· `paging_init` 拆桥 · 一图收尾
> - **推荐路径**：§1 总览 → §2 自解压 → §3 stext 裸机期 → §4 早期页表 → §5 跨越 MMU → §6 栈确立 → §7 重构页表 → §8 一图收尾

---

## 🎬 1. 两幕大戏与「双度开启 MMU」总览

一句话主线：ARM32 上，镜像在启动时会先后经历两幕"MMU 开关与栈生命周期"完全不同的戏。第一幕是 zImage 自解压阶段，临时开 MMU 加速解压；第二幕才是真正的内核本体，在完全无栈的裸机状态下手搭页表，再借一次性的 1:1 恒等映射跨入高位虚拟地址，完成物理 → 虚拟的不可逆跃迁，最终进入 C 语言的虚拟世界。

### 1.1 核心骨架总览

| | **第一幕：zImage 自解压** | **第二幕：内核本体启动** |
|---|---|---|
| 代码位置 | `arch/arm/boot/compressed/head.S` | `arch/arm/kernel/head.S` |
| 角色 | 微型解压引导程序 | 真正的操作系统本体 |
| 栈 | 链接脚本自带的私有临时栈 `.L_user_stack` | 起步完全无栈（禁止 `push/pop`），寄存器倒腾；开 MMU 成功后挂接正式 8KB 内核栈 |
| MMU / Cache | 临时创建 16KB 页表，将 4GB 空间全平铺 1:1 恒等映射（1MB Section） | 基于当前物理基地址现场算表，双重映射（1:1 物理过渡段 + 0xC0000000+ 高位线性段） |
| 目的 | 开启 D-Cache 极速解压内核（避免解压几 MB 内核卡死在冷 DDR 读写上） | 实现物理地址到高位虚拟地址的不可逆跃迁，进入 C 语言虚拟世界 |
| 终局 | 解压完毕 → 刷写 Cache → 关闭 Cache → 关闭 MMU → 现场全部废弃，跳向真实 Image 入口 `stext` | — |

### 1.2 交接契约

两幕之间通过跳转交接：`r0 = 0`、`r1 = arch_id`、`r2 = dtb_paddr`。

---

## 🧽 2. 阶段一：自解压阶段（zImage 解密）

### 2.1 为什么自解压要开 MMU

ARM 架构硬件规范中，**D-Cache 必须依赖 MMU 提供的内存属性（描述符里的 Cacheable / Bufferable 位）才能激活**。

若关 MMU 解压内核镜像，CPU 每天的字典匹配与解包写入都要直接打到物理 DRAM，解压数 MB 要耗时数秒；开启 MMU + D-Cache 后，解压速度提升一到两个数量级。

> [!NOTE] D-Cache 与 MMU 的硬依赖
> 「Cache 由 MMU 的属性位驱动」是 ARM 的硬件约束，不是软件实现细节——这正是自解压阶段即使只是加速也**必须先建表**的根因。细节见《ARM zImage head.S 启动 学习笔记》§8.6「为什么非得有表才能开 cache」。

### 2.2 `__setup_mmu`：16KB 表平铺 4GB（head.S:793-832）

位于 `arch/arm/boot/compressed/head.S`。页表就放在内核载入地址正下方 16KB 处：

```asm
__setup_mmu:
    sub     r3, r4, #16384          @ Page dir size（就在内核载入地址下方 16KB）
    bic     r3, r3, #0xff
    bic     r3, r3, #0x3f00
    mov     r0, r3
    mov     r9, r0, lsr #18
    mov     r9, r9, lsl #18         @ start of RAM
    add     r10, r9, #0x10000000    @ a reasonable RAM end

    mov     r1, #0x12               @ XN|U + section
    orr     r1, r1, #3 << 10        @ AP = 11
    add     r2, r3, #16384
1:  cmp     r1, r9                  @ if virt > start of RAM
    cmphs   r10, r1                 @   && end of RAM window
    bic     r1, r1, #0x1c           @ clear XN|U + C + B
    orrlo   r1, r1, #0x10           @ Set XN|U for 非 RAM 段
    orrhs   r1, r1, r6              @ Set RAM section（CB_BITS|0x12）
    str     r1, [r0], #4            @ 1:1 mapping
    add     r1, r1, #1048576        @ 每格 +1MB
    teq     r0, r2
    bne     1b
```

要点：

- **页表大小**：$4096\text{ 个条目} \times 4\text{ 字节} = 16\text{ KB}$。
- **映射粒度**：1MB Section 模式平铺。
- **映射范围**：从 `0x00000000` 到 `0xFFFFFFFF` 循环，将整个 4GB 虚拟地址全量等值映射到物理地址。
- **关键细节**：4GB 平铺是对的，但**只有 RAM 窗口 `[ram_start & ~1MB, +256MB)` 内的段才会置 Cacheable/Bufferable 位**；窗口外的非 RAM 段被强制 XN（不可执行）。注释明示这段映射会被之后的 `paging_init()` 移除。
- 紧随其后还有一段：把 PC 所在的 2MB 也按 cacheable 映射，专为"内核跑在 Flash 上"的场景准备（在 RAM 里跑则只是重复覆盖）。

### 2.3 会覆盖自身时跳过建表的前提（head.S:317-329）

```cpp
/*
 * Set up a page table only if it won't overwrite ourself.
 * That means r4 < pc || r4 - 16k page directory > &_end.
 */
...
orrcc   r4, r4, #1              @ remember we skip
blcs    cache_on
```

页表**不一定总能建立**：若 `r4 - 16K` 这块页表区会踩到正在执行的 zImage 本体，就放弃建表、放弃 `cache_on`，改用 `r4` 的最低位记录这件事；等 Image 复制完成（`relocate_kernel` 之后）再补做 `cache_on`（head.S:619-621）。所以"自解压必然带 Cache"不是恒成立的；页表可能根本不建，解压退回无 Cache 模式。

> [!WARNING] 先判表会不会压到自身，再决定是否开缓存
> 建表前务必核对 `r4 - 16K` 页表区是否会覆盖正在执行的镜像本体：会则**跳过建表与 cache_on**，并用 `r4` 最低位做记号，待 relocate 之后补做。这是自解压能安全兜底无 Cache 模式的前提——若忽略该判定强行建表，会直接砸中自身指令流。

### 2.4 退场：`cache_clean_flush → cache_off → __enter_kernel`（head.S:636-642）

```asm
mov     r0, r4                  @ start of inflate kernel
add     r1, r1, r0              @ end of inflated kernel
bl      cache_clean_flush
bl      cache_off
...
__enter_kernel:
    mov     r0, #0              @ must be 0
    mov     r1, r7              @ restore architecture id
    mov     r2, r8              @ restore atags pointer
ARM(    mov     pc, r4        ) @ call kernel（r4 = PHYS_OFFSET + TEXT_OFFSET）
```

准确退出顺序是：`cache_clean_flush` → `cache_off`（写 SCTLR 关 MMU + Cache）→ `__enter_kernel(r0=0, r1=arch_id, r2=atags/dtb)`；`r0` 在 `stext` 处会被复用，不能跨 `bl` 保存。若 bootloader 把 zImage 丢进 HYP 模式，会先 `__hyp_set_vectors` + `HVC #0` 降回 SVC 再进内核。

---

## 🏗️ 3. 阶段二：内核本体 `stext` 裸机期（无栈 + 就地热补丁）

### 3.1 入口约定与寄存器分工（head.S:72-173）

入参约定：`MMU = off`、`D-cache = off`、`I-cache = don't care`、`r0 = 0`、`r1 = machine nr`、`r2 = atags/dtb`。

`stext` 内部长期占用的寄存器：

| 寄存器 | 含义 |
|---|---|
| `r1` | machine id |
| `r2` | dtb / atags |
| `r4` | 页表物理基址（`__create_page_tables` 的返回） |
| `r5` | procinfo 高字（LPAE） |
| `r8` | `PHYS_OFFSET`（= `__pa(_text) - TEXT_OFFSET`） |
| `r9` | cpuid（MIDR） |
| `r10` | procinfo |
| `r13` | `__turn_mmu_on` 之后要跳的"虚拟地址" |

`safe_svcmode_maskall r9` 先把模式/中断按住；`mrc c0, c0` 取 MIDR → `__lookup_processor_type` 在 `__proc_info` 链里线性匹配，`r5 = 0` 直接 `__error_p`。LPAE 内核还会读 `ID_MMFR0[3:0]` 校验 VMSA 支持。

### 3.2 `CONFIG_SMP_ON_UP`：`__fixup_smp` 单核/多核自适应与就地热补丁

为了实现同一份多核内核镜像在单核芯片上无缝跑满性能，内核在最早期的物理地址运行期执行探测（探测链全部用 `r9 = MIDR`）：

1. `MIDR[23:20] != 0xF` → 非 ARMv7 语义，直接判 UP（`bne __fixup_smp_on_up`）。
2. `MIDR & 0xff00fff0 == 0x4100b020` → 是 ARM11MPCore（首款多核老祖宗），`reteq lr` 认定 SMP 走人。
3. `mrc p15, 0, r0, c0, c0, 5` 读 `MPIDR`，仅当 `MPIDR[31:30] == 0b10` 才承认多核扩展，否则判 UP。
4. 是 Cortex-A9（`0x4100c090`）时再做一层针对性探测：`mrc p15, 4, r0, c15, c0` 取 SCU（Snoop Control Unit）基址；若基址为 0 则直接判 UP（典型：Aegis 单核 SoC 未挂 SCU）；否则 `ldr r0, [r0, #4]`（SCU Config）`& 0x3` 取核数，为 1 才判 UP。

确认单核后执行热补丁（`__do_fixup_smp_on_up`）：

```asm
__do_fixup_smp_on_up:
    cmp     r4, r5
    reths   lr
    ldmia   r4, {r0, r6}                    @ r0 = 相对 __smpalt_begin / r6 = 替换指令
ARM(    str     r6, [r0, r4]            )   @ 直接 str 就地覆写代码段
THUMB(  add     r0, r0, r4              )
    add     r4, r4, #8                      @ 每项 8 字节 [offset, replacement opcode]
THUMB(  strh    r6, [r0], #2            )   @ Thumb-2 拆两半字写，抗非对齐
THUMB(  strh    r6, [r0]                )
    b       __do_fixup_smp_on_up
```

> [!NOTE] Cortex-A9 的 SCU 基址兜底
> 对 Cortex-A9，核数判定依赖读 SCU（Snoop Control Unit）。该族某些单核量产 SoC（如 Aegis）并未挂接 SCU，基址读出为 0——此时**直接判单核是稳健落点**，避免对不存在的外设做解引用而读到无效核数。这是针对实测目标的兜底探测，而非通用规则，其他核仍以 MPIDR 为准。

要点：

- 链接脚本通过 `include/asm/alternative.h` 的 `ALT_SMP / ALT_UP` 宏生成补丁条目，落在 `.alt.smp.init` 段，由 `arch/arm/kernel/vmlinux.lds.S` 收拢为 `__smpalt_begin / __smpalt_end`。
- 每条补丁 8 字节结构：`[Offset, Replacement Opcode]`。汇编引擎逐条遍历，直接用 `str` / `strh` 将代码段中的多核总线屏障指令（如昂贵的跨核 `dmb ish`）就地改写为单核轻量指令或 `nop`。
- 一致性由调用方的常规 `flush_icache_range()` 负责，因此补丁过程本身无需再刷 I-Cache 或 `isb`。
- 顺带：`smp_on_up` 变量（head.S:570-574）用 `ALT_SMP(.long 1) / ALT_UP(.long 0)` 被同一机制改成 0，供 C 代码运行期查询。

### 3.3 `__fixup_pv_table`（`CONFIG_ARM_PATCH_PHYS_VIRT`）

`ARCH_MULTIPLATFORM` 下 `PHYS_OFFSET` 编译期未知，链接器把 `__va / __pa` 用到的常数留在 `__pv_table`，此处在裸机期就地改掉。IMM8 / IMM16 指令改写与上面的 SMP 补丁属于同一类 "live patching"。

---

## 🗺️ 4. 阶段三：`__create_page_tables` —— 现场手搭早期页表（head.S:187-377）

```asm
__create_page_tables:
    pgtbl   r4, r8                      @ r4 = r8 + TEXT_OFFSET - PG_DIR_SIZE（页表的物理地址）
```

`pgtbl` 宏（head.S:67-70）就是在给 `__phys_to_virt` 打补丁前，先把 `swapper_pg_dir` 换算成物理地址。`Image` 文件编译时并不包含实质的页表内容，仅在链接脚本中留出代码段前 `PG_DIR_SIZE` 的占位空洞（非 LPAE `0x4000`，LPAE `0x5000`），真正的表内容运行时现场算。

流程（严格按代码顺序）：

1. **清空 PG_DIR_SIZE**：`str` 4 连发循环写 0（head.S:193-201）。非 LPAE 16KB，LPAE 20KB。
2. **LPAE 才有**：先建 4 个 PGD 项指向紧跟其后的 4 张 PMD 表（`orr r3, r3, #3` = block 类型 + `L_PGD_SWAPPER`）。
3. **取该 CPU 的 section 属性**：`ldr r7, [r10, #PROCINFO_MM_MMUFLAGS]`。
4. **1:1 恒等映射**（head.S:233-246）：只覆盖 `__turn_mmu_on .. __turn_mmu_on_end` 所跨的 section；注释明示 *"This identity mapping will be removed by paging_init()"*。
5. **高位线性映射（"the main matter"）**（head.S:249-278）：以 `r8 = PHYS_OFFSET`、`r7 = MMU flags` 为基，从 `KERNEL_OFFSET >> SECTION_SHIFT` 槽位起，把整个内核镜像（`ldr r6, =(_end-1)`）逐 1MB 铺到 `0xC0000000+`；同时把物理起止写入 `kernel_sec_start / kernel_sec_end`，供后面 C 侧 `map_lowmem()` 用来"挖掉"内核自身占用的 memblock 区间。
6. **XIP 分支**：内核不在 RAM 时另建一份 `XIP_VIRT_ADDR` 映射。
7. **DTB 映射**（head.S:298-309）：把 `r2` 所指地址映射进 `FDT_FIXED_BASE` 对应槽，映射 2 个 section 以防跨 1MB 边界。
8. **`CONFIG_DEBUG_LL`**：`addruart` 造一个串口 IO 段映射并强制 XN；本机 DEBUG_LL 未开 → 无此项。另有 NetWinder / RiscPC 的历史遗留映射。

> [!NOTE] 早期只建两块映射，正是「田忌赛马」
> 本阶段刻意只做两件事：给 `__turn_mmu_on` 一小段铺 1:1 浮桥，再给内核镜像铺高地位。既满足「点火那一纳秒流水线仍取低地址指令」的安全，又不浪费表项铺全 4GB——这是阶段一（zImage 平铺 4GB）与阶段二（内核本体按需映射）的根本分野。

---

## 🔌 5. 阶段四：跨越 MMU 虫洞 —— `__enable_mmu` 与 `__turn_mmu_on`（head.S:467-519）

```asm
__enable_mmu:
    bic     r0, r0, #CR_A            @ 对齐陷阱开关（ARMv6+ 默认已无对齐访问）
    ; ...（DCACHE/BPredict/ICACHE_DISABLE 时清 CR_C/CR_Z/CR_I 等位）
    mov     r5, #DACR_INIT
    mcr     p15, 0, r5, c3, c0, 0    @ load domain access register
    mcr     p15, 0, r4, c2, c0, 0    @ load page table pointer = 物理页表址
    b       __turn_mmu_on

    .align  5
    .pushsection    .idmap.text, "ax"
ENTRY(__turn_mmu_on)
    mov     r0, r0
    instr_sync
    mcr     p15, 0, r0, c1, c0, 0    @ 硬件合闸：MMU + Cache 正式通电
    mrc     p15, 0, r3, c0, c0, 0    @ read id reg（把 mcr 的效果冲刷进流水线）
    instr_sync
    mov     r3, r3
    mov     r3, r13                  @ r13 不是栈，是链接期算好的"虚拟地址"
    ret     r3
__turn_mmu_on_end:
ENDPROC(__turn_mmu_on)
    .popsection
```

在 `__enable_mmu` 之前，`stext` 里 `ldr r12, [r10, #PROCINFO_INITFUNC]; add r12, r12, r10; ret r12` 已经调用了 `__v7_setup`（`PROCINFO_INITFUNC`），它才是 TTBCR/TTBR1 的真正拥有者：

```asm
@ arch/arm/mm/proc-v7-2level.S:146-153（非 LPAE）
mcr     p15, 0, \zero, c2, c0, 2          @ TTBCR = 0 → T0SZ = 覆盖 4GB（只用 TTBR0）
ALT_UP( orr \ttbr0l, \ttbr0l, #TTB_FLAGS_UP)   @ 顺带给 page table memory 属性
mcr     p15, 0, \ttbr1, c2, c0, 1         @ load TTB1（装载了但此刻用不上）
```

`r8 = r4`（`mov r8, r4 @ set TTBR1 to swapper_pg_dir`，head.S:167）—— 交给 `__v7_setup` 的 TTBR1 是物理地址，真正生效要等 `cpu_init` 重装。**结论：早期只有 TTBR0 干活，TTBR1 实为闲置。**

### 5.1 为什么硬件合闸与跳板不会死机

1. **合闸瞬间的 1:1 浮桥掩护**：写完 `mcr` 使能 MMU 后的那一纳秒，CPU 取指流水线仍在低地址抓取接下来的 `mrc`、`mov` 指令。此时 MMU 已生效，但低地址被第 4 步的 1:1 段兜住，虚拟查询返回等值物理地址，翻译结果等于物理地址，流水线平滑无感执行，绝不触发 Prefetch Abort。这正是 `__turn_mmu_on` 被单独塞进 `.idmap.text` 并 `.align 5`（32 字节对齐、不跨段）的原因。
2. **`ret r3` 绝对跳转**：`r13` 此时并不作为栈，而是携带编译期由 `vmlinux.lds` 固化的虚拟高位入口符号。`ret r3`（`bx r3`）把 `0xC000xxxx` 拍进 PC，同步冲刷作废 CPU 流水线中预取的所有物理指令；重新拉取的指令全面命中高位线性映射段，CPU 彻底跃迁进入虚拟大陆。
3. **`r13` 的两种人格**：`__secondary_switched` 处 `mov r13, r12` 同理（head.S:426），所以汇编期 `sp` 全程只是临时寄存器。

> [!WARNING] 合闸瞬间不能脱离 1:1 浮桥
> 使能 MMU 与真正 `ret r3` 之间，PC 仍在物理低地址执行——这段**必须**落在 1:1 恒等段覆盖的 `__turn_mmu_on .._end` 之内。若该段被删或跨了对齐边界，合闸那一瞬流水线取指即 Prefetch Abort。这解释了为什么把它钉在 `.idmap.text` 且 `.align 5`（不跨段）。

---

## 🛗 6. 阶段五：虚拟世界着陆 —— 栈正式确立（这块栈属于 idle 任务）

### 6.1 主核：`__mmap_switched`

`__mmap_switched` 在 `arch/arm/kernel/head-common.S`（head.S:602 处 `#include` 进来），不在 head.S 正文：

```asm
__mmap_switched:                    @ __INIT 段，非 PIC
    mov     r7, r1
    mov     r8, r2
    mov     r10, r0
    adr     r4, __mmap_switched_data
    mov     fp, #0
ARM(    lmia    r4!, {r0, r1, sp}   )   @ r0=__bss_start, r1=__bss_stop, sp=正式内核栈顶
    sub     r2, r1, r0
    mov     r1, #0
    bl      __memset                    @ 清 .bss（必须在 1:1 被拆掉之前完成）
    adr_l   r0, init_task               @ ★ swapper/0 —— CPU0 的 idle 任务
    set_current r0, r1                  @ 写 current 指针
    ldmia   r4, {r0, r1, r2, r3}
    str     r9, [r0]                    @ processor_id
    str     r7, [r1]                    @ __machine_arch_type
    str     r8, [r2]                    @ __atags_pointer
    strne   r10, [r3]                   @ cr_alignment
    mov     lr, #0
    b       start_kernel
```

**「第一块栈 = idle 任务」的证据链：**

```c
/* include/asm-generic/vmlinux.lds.h:425-432 */
__start_init_stack = .;
init_thread_union = .;        /* 同一个 8KB 块的前 THREAD_SIZE 是栈、末尾是 thread_info */
init_stack = .;
. = __start_init_stack + THREAD_SIZE;
__end_init_stack = .;
```

```c
/* arch/arm/include/asm/thread_info.h:21-31 */
THREAD_SIZE_ORDER = 2 (KASAN) / 1 (默认)   /* 本机非 KASAN：THREAD_SIZE = 4K<<1 = 8KB */
THREAD_START_SP = THREAD_SIZE - 8          /* 栈顶（满递减，留 8 字节） */
THREAD_ALIGN = 2*THREAD_SIZE (VMAP_STACK)  /* 本机 VMAP_STACK=y，按 16K 对齐以隔出 guard page */
```

```c
/* init/init_task.c:111 */
.stack = init_stack            /* init_task 的 task_struct->stack 正是这块 */
```

`init_task` 就是 swapper/0，即 CPU0 的 idle 任务：`start_kernel()` 把它作为 `init_task` 一路带到 `rest_init()` → `cpu_idle()/do_idle()`，`current` 在无进程可调度时就是它。`set_current` 那一行在 C 世界还没醒来的时候就把 `current → init_task` 钉死，所以内核建立的第一块栈从生到死都是 idle 任务的栈，`init_stack` 也是 pid=0 的栈。

### 6.2 从核：`__secondary_switched`

`secondary_data` 的准备（`arch/arm/kernel/smp.c:131-156`，`__cpu_up(cpu, idle)`）：

```c
secondary_data.stack = task_stack_page(idle) + THREAD_START_SP;  /* 每核第一块栈 = 该核 idle 任务栈 */
secondary_data.pgdir = virt_to_phys(idmap_pgd);                   /* 从核用另一张专用 1:1 表 */
secondary_data.swapper_pg_dir = get_arch_pgd(swapper_pg_dir);     /* 真正的虚拟 swapper_pg_dir */
secondary_data.task = idle;
sync_cache_w(&secondary_data);                                    /* 从核此时 D-cache 可能关着，须脏线冲刷 */
```

从核入口 `secondary_startup`（head.S:388-431）用 `ldrd r4, r5, [r3]` 取 `idmap_pgd`（`mov_l r12, __secondary_switched` 走 GOT 取绝对址，因为从核在别的物理位置执行）。注意它装载的是 `idmap_pgd` 而非主核那张早期表 —— `init_static_idmap()`（arch/arm/mm/idmap.c:113-127）在 `early_initcall` 里用 `pgd_alloc()` 建好的静态 1:1 表，覆盖 `__idmap_text_start..__idmap_text_end`。

着陆点（head.S:434-447）：

```asm
ENTRY(__secondary_switched)
#if defined(CONFIG_VMAP_STACK) && !defined(CONFIG_ARM_LPAE)
    mrc     p15, 0, ip, c2, c0, 1   @ read TTBR1    ← 先切回 swapper_pg_dir，
    mcr     p15, 0, ip, c2, c0, 0   @ set TTBR0       因为 idmap 不覆盖 vmalloc 区
    instr_sync
#endif
    adr_l   r7, secondary_data + 12
    ldr     sp, [r7]                @ ★ 分水岭：挂上本核 idle 任务栈顶
    ldr     r0, [r7, #4]            @ r0 = secondary_data.task（idle 的 task_struct）
    mov     fp, #0
    b       secondary_start_kernel
```

> [!TIP] 从核为何先 TTBR1→TTBR0 切表
> 从核用专用 `idmap_pgd` 只覆盖 `__idmap_text`，不覆盖 vmalloc 区；而 `VMAP_STACK=y` 下 C 栈很可能落在 vmalloc——着陆后一进 C 就会访到未映射区。故 `__secondary_switched` 里先读回 TTBR1、写回 TTBR0（切到 `swapper_pg_dir`）再挂栈、`b secondary_start_kernel`，属把后续 C 世界所需的虚拟映射先就位。

### 6.3 AAPCS 秩序自此恢复

- `r0 ~ r3` 传前 4 个参数，多余压栈；`r4 ~ r11` callee-saved，函数头 `push {r4-r11, lr}`；`r12` ip、`r13` sp、`r14` lr、`r15` pc。
- `__mmap_switched` 里那个 `mov lr, #0` 是给 `start_kernel()` 的返回地址清零 —— 从此栈回溯有终止标志。

---

## 🧱 7. 阶段六：C 语言重构页表与「过河拆桥」（`paging_init`）

调用链：`start_kernel() → setup_arch() → paging_init(mdesc)`（`arch/arm/mm/mmu.c`），顺序为：

```c
prepare_page_table();        /* ★ 拆桥：pmd_clear 掉内核镜像以下全部映射 */
map_lowmem();                /* 按 memblock + kernel_sec_start/end "挖洞" 映射 DRAM */
memblock_set_current_limit(arm_lowmem_limit);
map_kernel();                /* 用 section 把内核 RAM 切成 X 段 / NX 段 */
dma_contiguous_remap();
early_fixmap_shutdown();
devicemaps_init(mdesc);
kmap_init();
tcm_init();
top_pmd = pmd_off_k(0xffff0000);
zero_page = early_alloc(PAGE_SIZE);
bootmem_init();
__flush_dcache_folio(NULL, page_folio(empty_zero_page));
```

### 7.1 拆桥 = `prepare_page_table()`（mmu.c:1273）

```c
for (addr = 0; addr < MODULES_VADDR; addr += PMD_SIZE)   /* KASAN 时拆成两段跳过 shadow 区 */
        pmd_clear(pmd_off_k(addr));
for ( ; addr < PAGE_OFFSET; addr += PMD_SIZE)
        pmd_clear(pmd_off_k(addr));                       /* XIP 时先跳过 _exiprom */
```

- 低位 `0 ~ MODULES_VADDR (= PAGE_OFFSET - 16M)` 被清空 → 阶段四第 4 步建的 1:1 恒等桥在此注销；
- `MODULES_VADDR ~ PAGE_OFFSET` 留给模块区；
- 冲刷在 `mmu.c` 的 `local_flush_tlb_all()`（**本地**，不是全核广播）；`map_kernel / create_mapping` 的属性变更同理走 `local_flush_tlb_kernel_range`。
- **为什么不崩**：此时 `PC`、`SP`、数据段全部扎根在 `0xC0000000+` 高位空间，改写低地址 PMD 槽位只是在 `swapper_pg_dir` 这个数组里改一个 4 字节数据；`pmd_off_k()` 走的是高位索引，不依赖被删除的映射，对高地址执行流没有任何扰动。

> [!WARNING] 拆桥前必须已脱离低地址
> 「拆桥」只在对**高地址虚址**安全之后才合法：清除的是低位段映射（1:1 桥与用户区），而 `PC/SP/数据` 必须全部已迁至高位的 0xC0000000+ 空间。若拆桥时仍有任何执行流依赖低物理地址映射（如自解压壳刚进 `stext` 的瞬间），`pmd_clear` 会造成 Prefetch/Data Abort——这正是 §5 的 1:1 桥到 §7.1 才能拆的时间窗口。

### 7.2 精细程度的实话（mmu.c:920-986、1537-1567）

- `create_mapping()` 的注释写着 *"we take full advantage of sections and supersections"*：**不是清一色 4KB PTE**。`mem_types[].prot_l1 = PMD_TYPE_TABLE` 时经 `alloc_init_p4d → … → create_pte_mapping` 才落 4KB 表；能按 1MB 对齐的仍用 section；物理地址 ≥4GB 走 supersection（16MB）；LPAE 走另一套。
- `map_kernel()` 只做 section 级 RWX 切分，分界是 `round_up(__pa(__init_end), SECTION_SIZE)`：之前 `X`，之后 `NX`。它自己的注释承认：*"Notice that we are dealing with section sized mappings here … some of the non-executable part of the kernel memory is actually mapped as executable. This will only persist until … we remap the whole kernel with page granularity."*
- 真正页粒度的权限收紧发生在更晚的 `create_mapping_late()`（mmu.c:988，目前主要被 `kernel/efi.c` 用于 EFI 区域）与 module / `set_memory_*` 路径。
- 所以 `.text RO+X / .rodata RO+NX / .data+栈 RW+NX` 作为**最终形态**描述可以，但 **`paging_init()` 当场**给出的是 section 粒度的近似。

> [!NOTE] 「最终」是页粒度，此刻是 section 粒度
> `paging_init()` 当场交付的是 1MB section 的近似权限（`__init_end` 为 X/NX 分界），真正的页级 RO/NX 收紧要更晚才发生。把 `.text/.rodata/.data` 的 RO/NX 当作 `paging_init` 当下的状态去核对映射，会与实测不符——它描述的是最终形态。

### 7.3 空间腾出与双 TTBR 的真实分工

- 低位 `0 ~ TASK_SIZE` 清空，专为用户态保留。
- **非 LPAE（本机）**：`TTBCR.T0SZ = 0` ⇒ TTBR0 覆盖整 4GB、TTBR1 闲置。内核高地址段之所以在每个用户进程里都可见，是因为内核的 PMD/PTE 表指针被**复制**进每份 `mm->pgd` 的高位槽（`arch/arm/mm/context.c` 的 `check_and_switch_context / copy_pmd_table` 一套），而不是靠 TTBR1。
- **LPAE**：`v7_ttb_setup`（proc-v7-3level.S:117-136）才真正用 `TTBR1_SIZE(T1SZ)` 做分工，且带条件 —— `cmp \ttbr1, \tmp, lsr #12` 判断 `PHYS_OFFSET > PAGE_OFFSET?`，只有 `PHYS_OFFSET <= PAGE_OFFSET` 时才 `orrls` 打开 T1SZ，否则从核的 idmap 会被 TTBR1 吃掉。
- 从核 / 休眠用的是独立 `idmap_pgd`（`virt_to_idmap()`），主核早期表被拆掉之后 idmap 仍长期存在，专门服务 `cpu_resume`、`__cpu_soft_restart`、`secondary_startup`。

---

## 📈 8. 一图收尾

```
stext ─ safe_svcmode_maskall ─ MIDR→__lookup_processor_type ─ r8=PHYS_OFFSET
  ├ __vet_atags            （校验 r2：ATAG_CORE / DTB magic，不合法则清 0）
  ├ __fixup_smp            （MIDR/MPIDR/SCU → 就地 str 覆写 .alt.smp.init 表项；无 icache 维护）
  ├ __fixup_pv_table       （多平台 P:V 常数补丁）
  ├ __create_page_tables   （清 16KB → 1:1 for __turn_mmu_on → 0xC000xxxx 内核段 → DTB 2 sections → [DEBUG_LL]）
  ├ PROCINFO_INITFUNC = __v7_setup （出 SCTLR 值 r0；写 TTBCR=0 / TTBR0 / TTBR1）
  └ __enable_mmu（DACR+TTBR0）→ __turn_mmu_on(.idmap.text) → ret r13
                        │
           __mmap_switched：装 init_thread_union+THREAD_START_SP（= CPU0 idle / swapper-0 的 8KB 栈）
                             → 清 .bss → set_current init_task → start_kernel
           secondary_startup：装载 idmap_pgd → __turn_mmu_on → __secondary_switched
                             → 先 TTBR1→TTBR0 切表（VMAP_STACK）→ 装该核 idle 任务栈 → secondary_start_kernel
           setup_arch → paging_init：prepare_page_table(拆 1:1) → map_lowmem → map_kernel → … → local_flush_tlb_all
```

> 中断相关内容（中断入口与分发、下半部 SoftIRQ 等）见同目录《kernel 中断系统.md》。

---

## 🔗 思考与延伸索引

- [[ARM zImage head.S 启动 学习笔记]] — 两幕大戏的第一幕：自带重定位/页表/malloc 的自解压壳如何铺 4GB 表、解压并交棒 `stext`
- [[ARM zImage head.S 启动 学习笔记]]（§8.6/§9）— 与 `__create_page_tables` 同源的建表与 cache 属性机理
- 《kernel 中断系统.md》（同目录）— `stext` 中断入口与分发、下半部 SoftIRQ 等衔接
- `arch/arm/mm/mmu.c` — `prepare_page_table` / `map_lowmem` / `map_kernel` / `create_mapping_late` 页粒度收紧
- `mm/idmap.c` + `init_static_idmap` — 从核/休眠专用 `idmap_pgd` 的长期生命周期

---

## 📚 参考资料

1. **Linux 内核源码** — *arch/arm/kernel/head.S* 与 *head-common.S*（`stext`、`__create_page_tables`、`__enable_mmu`/`__turn_mmu_on`、`__mmap_switched`/`__secondary_switched`）.
2. **Linux 内核源码** — *arch/arm/mm/mmu.c*（`paging_init`、`prepare_page_table`、`map_lowmem`、`map_kernel`、`create_mapping_late`）.
3. **Linux 内核源码** — *arch/arm/mm/proc-v7-2level.S* / *proc-v7-3level.S*（`__v7_setup`、TTBCR/TTBR 设置）与 *mm/idmap.c*.
4. **Linux 内核源码** — *arch/arm/kernel/smp.c*（`__cpu_up`、`secondary_data`）与 *include/asm-generic/vmlinux.lds.h* / *arch/arm/include/asm/thread_info.h*.
5. **Linux 内核** — *include/asm/alternative.h*（`ALT_SMP/ALT_UP`）与 *vmlinux.lds.S*（`__smpalt_begin/__smpalt_end`）.<br/>
6. **ARM** — *ARM Architecture Reference Manual (ARMv7-A/R)*（`__v7_setup` 涉及的 SCTLR、TTBCR/TTBR、DACR 字段）.
