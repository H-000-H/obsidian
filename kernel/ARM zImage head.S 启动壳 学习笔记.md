# ARM zImage head.S 启动壳 学习笔记

> 基于 `arch/arm/boot/compressed/head.S`，Linux lf-6.18.y，IMX6ULL / Cortex-A7

---

## 一、zImage 启动壳主线

### 0. 一句话总纲

zImage 是一个**自带重定位、自带页表、自带 malloc** 的 5MB 自举壳，毕生事业：把 piggy 里的压缩内核解压到 `r4`，清好 cache，跳进去自杀。所有指令只伺候三块地：

- `[r4, r4+r9)` —— 内核将住的地
- `[restart, r10]` —— 壳现在占的地（text + piggy + DTB + BSS + stack + 64K heap）
- `GOT` —— 链接器"平行世界"的电话簿

### 1. PIC 三板斧（全文反复出现的母题）

镜像可能被放在任意物理地址，绝对寻址失效：

```asm
adr   \res, .Llabel      @ ① PC 相对拿"锚点"真实地址
ldr   \tmp, [\res]       @ ② 读链接期算好的【相对差值】（平移不变，永远对）
add   \tmp, \tmp, \res   @ ③ 锚点 + 差值 = 目标真实地址
```

**应用：**

- `get_inflated_image_size`（156）：定位 piggy 末尾 4 字节 = 解压后大小（构建系统追加，u32 小端，最大 4GB）；`ldrb×4 + orr 移位拼装` = 手工 `get_unaligned_le32`（地址可能非 4 字节对齐，MMU 未就绪前不能冒险做非对齐访问）；
- `LC0/LC1` 常数表（660）：`.word LC0` 存自己链接地址 → 与 `adr` 真实地址一比得 `delta = 真实 − 链接`；其余项存差值还原 `_edata`、`__bss_start`、`_end`、GOT 边界、栈顶。

### 2. 字节序

- `be32tocpu` / `rev_l`：ARMv6+ 单条 `rev` 翻转 4 字节（v5 用 eor/bic/ror 模拟）；大端内核（`__ARMEB__`）下 no-op。
- DTB magic 字段强制 big-endian：`OF_DT_MAGIC` 按端序定义成 `0xd00dfeed` / `0xedfe0dd0`，比较寄存器数值。

### 3. 入口结构

```asm
.section ".start","ax"         ← 链接在镜像最前，start: = 第一条指令
8×nop（=32B）                  ← 骗老式 a.out bootloader 的垫片；Thumb2 只留 7 + sub pc,pc,#3 切态
b 1f                           ← 跳过镜像头数据：
                                  magic 0x016f2818 / _magic_start / _endian / _table
                                  _magic_table → .table{6, "KSLZ", size 字段偏移,
                                  _kernel_bss_size, TEXT_OFFSET, MALLOC_SIZE}（自描述，外部工具读）
1: setend / mrs r9,cpsr        ← r9 = boot CPSR（后进 SPSR，内核用它恢复 boot mode）
   r7=r1（DTB 指针） r8=r2（atags）   ← U-Boot 寄存器传参（new-ABI：r0=0, r2=0, r3=0）
   Angel 检测：tst cpsr,#3，User 模式才走 swi 0x123456（半主机仿真器的历史遗留，实机跳过）
safe_svcmode_maskall           ← 真正的进 SVC + 关 IRQ/FIQ
bl __hyp_stub_install          ← HYP mode（ARMv7 VE=EL2 前身）时留 hyp stub
```

**arch-specific linker insertion（256 注释）**：head-sa1100.S 等也声明 `.start`，链接器把 `*(.start)` 拼在 `*(.text)` 前 → 平台补丁代码恰好落在 `.text` 指令处、靠 fall-through 执行；须保 r7/r8/r9。

### 4. 决定内核落点 r4

```asm
CONFIG_AUTO_ZRELADDR:  r4 = (pc & 0xf8000000) + TEXT_OFFSET   @ 猜 RAM base
                       → bl fdt_check_mem_start 用 DTB memory 节点校验/改算
非 AUTO:               r4 = zreladdr（编译期写死）
```

- **AUTO 的枷锁**：zImage 必须在 RAM base 起第一格 128MB 内，且 base 最好 128MB 对齐；base 非 0x80000000 也行（任意 128MB 倍数），不对齐则靠 DTB 救（`fdt_check_mem_start`：mask 值落在 memory 区间 → 保留；否则用 DTB 最低 base，round 2MiB；kdump `usable-memory-range` 优先）。追加 DTB 时跳过校验（其 memory 节点不可信）。
- **非 AUTO**：地址自由，但一个镜像只配一种布局。

### 5. 寄存器账本（restart 前后）

| reg | 含义 |
|-----|------|
| r4  | 内核最终地址（LSB=1 记号：曾跳过 cache_on，621 行 bic 还原） |
| r5  | 追加 DTB 大小（检测后） |
| r6  | `_edata` 真实地址（检测后已推过 DTB） |
| r9  | 解压后大小（第 11 步可能虚增，以掩护 DTB） |
| r10 | 镜像地盘尾（bss + stack + heap） |
| sp  | 栈顶；heap = [sp, sp+64K) 向上，栈向下，背对背 |

### 6. appended DTB 与 ATAGs 合并（ATAG_DTB_COMPAT）

流程步骤 7~12：

7. 读 `[r6,#0]` 查 magic → 读 `[r6,#4]` totalsize（`be32tocpu`）；
8. 扩容 1.5×、clamp [32K, 1MB]，临时 `sp += r5`（DTB 物理压在 BSS 头上，调 C 前躲开）；
9. `atags_to_fdt(r8, r6, r5)`；返回 1 → 盲猜 ATAGs 在 RAMbase+0x100 重试（`r4 − TEXT_OFFSET` 清 LSB + 0x100）；
10. `r8 = r6`（今后用合并后的 appended DTB）；
11. `r9` 虚增 `max(0, _kernel_bss_size − (_edata − wont_overwrite))`：解压产物 r9 不含内核 BSS，BSS 会向上多咬一截，防它碾死 DTB；
12. 重读真实 totalsize，`r6/r10/sp` 永久 `+r5` 划地盘。

### 13. 撞车判定与 relocate

**豁免（任一成立即原地解压）：**

- ① `r4 ≥ r10 + 16K`：内核整个在我们头顶（16K = cache_on 页表在 r4−16K）
- ② `r4 + r9 ≤ &wont_overwrite`：内核整个塞进脚下"已执行完的死代码区"

`wont_overwrite` 之上 = 活物（剩余代码 + decompress 函数 + piggy 压缩源），警戒线不许碰；它以下是已执行完的 head 前奏，砸了无所谓。

**撞了 → 搬家：**

- 新家 = `r4 + r9 +`（reloc 代码长，256 对齐）← 躲到内核屋顶，且不被自己代码尾踩
- 拷贝 `[restart↓ .. _edata+DTB]`，从尾到头（`ldmdb/stmdb`，倒序防重叠覆盖）
- 不拷 BSS/栈/heap（脏数据，新家重建）；ATAGs 已折进 DTB 随之走
- `mov pc, r0` → 重入 restart：全部坐标基于新地址重算（这就是 restart 存在的意义）

> 注意：搬的永远是壳，`r4` 不动；内核不是"被搬"，是"壳先躲开再压进去"。（真 kernel 的 `relocate_kernel` 是另一回事。）

### 14. GOT fixup + BSS（壳的 C 运行时铺床）

```asm
adr LC0 → r1 = LC0 链接值 → sub → delta
delta == 0 且 r5 == 0 → not_relocated（白捡）
否则：
   r11/r12（GOT）、r2/r3（BSS） += delta
   GOT 逐项 += delta；落在 [r2,r3) 的再加 += r5（BSS 被 DTB 挤后移）
   条件短路链：cmp / cmphs / addhi == if(a<=x && x<b)
清 BSS：str r0,[r2],#4 ×4 循环到 r3（= 真实 __bss_start.._end，跳过 DTB）
补做 cache_on（tst/bic/blne r4 的 LSB 记号）
```

两层 BSS 各清各的：壳的 BSS 这里清；内核的 BSS 由 stext 进门自清（r9 不含它）。

### 15. 解压与交棒

```asm
r0 = r4, r1/r2 = heap[sp, sp+64K), r3 = r7
bl decompress_kernel            @ zlib 等要 malloc：misc.c bump allocator（free_mem_ptr）
get_inflated_image_size         @ 重读尾部 4 字节（r9 早被征用；piggy 是唯一权威）
r1 = r4 + size → cache_clean_flush  @ 写脏的内核必须刷回 RAM（cache_on 过！）
cache_off → HYP re-entry 判断 → __enter_kernel → bx r4
```

壳占的 RAM 无需"释放"：内核从 DTB memory 节点管 memblock，那 5MB 自然回收覆盖。

### 16. XIP（CONFIG_ZBOOT_ROM）对照

|          | PIC（你的板）                 | XIP                                                  |
|----------|-------------------------------|------------------------------------------------------|
| relocate | 会，自动躲车                  | 无（text 锁在 flash）                                |
| GOT/BSS 指针 | 全部 += delta（+BSS 项 += r5） | BSS 项不修（住 ZBSSADDR 定死位），其余 += delta    |
| 清 BSS   | 要                            | 照样要（地址对 ≠ 内容是零）                          |
| 撞车     | 自救                          | 无救济：静默压毁 BSS/栈 → hang，只能靠人工排布 zreladdr/BSS/heap/DTB 不重叠 |

### 17. 常见坑速查

- AUTO 下把 zImage 放到 base+128MB 外 → r4 算出错误地址，无声死机；
- 追加 DTB 且 base 非 128MB 对齐 → 校验被跳过，必死；
- 忘追 piggy 尾 4 字节（自制镜像）→ r9 拿到垃圾值，豁免/relocate 全错；
- relocate 新家不留 reloc 代码余量 → 跳回执行被覆盖的半条指令；
- cache_on 过而 clean_flush 范围用错 size → 内核残缺，bx r4 炸；
- blo/bhs 无符号 vs blt/bge 有符号，地址比较一律前者。

---

## 二、ARMv7 CP15 速查表（compressed/head.S 收官笔记）

> 目标：i.MX6ULL / Cortex-A7 / zImage 解压壳

### 0. 一句话总纲

32 位 ARM 启动 = 把数据摆成硬件查询的形状（页表），把指针和控制位塞进 CP15，最后 ISB 冲掉旧世界的流水线。

### 1. 指令格式

```asm
mcr/mrc p15, <op1>, <Rt>, <CRn>, <CRm>, <op2>
                 寄存器号 ──┘        └── 动作编号（查表件，没人背）
```

### 2. 柜号地图（CRn，值得记的只有这几个）

| 柜号 | 寄存器     | 作用                          | 出现位置举例                       |
|------|------------|-------------------------------|------------------------------------|
| c0   | MIDR       | 我是谁（call_cache_fn 查表分发） | `mrc c0,c0` → proc_types 匹配    |
| c0   | ID_MMFR0   | 我有没有 VMSA MMU             | [3:0]==1 → 才建表                 |
| c1   | SCTLR      | 总开关                        | 开 I/D cache + MMU 全在这一条     |
| c2   | TTBR0      | 页表基址                      | `mcr c2,c0,0` ← r3（__setup_mmu 的表） |
| c2   | TTBCR      | 表分割/N=0 只用 TTBR0         | `mcr c2,c0,2`                     |
| c3   | DACR       | 权限域（domain0=client）      | `mcr c3,c0,0` ← 0xfffffffd        |
| c7   | cache 操作 | clean/inval/DSB/ISB           | `c7,c10,4`=drain; `c7,c5,4`=ISB   |
| c8   | TLB 操作   | flush I/D TLB                 | `mcr c8,c7,0`                     |

### 3. SCTLR 位号（唯一值得背的 bit 表）

| bit   | 名  | 作用                                            |
|-------|-----|-------------------------------------------------|
| bit0  | M   | MMU 开 ← 唯一点火位，一写入即所有访存开始查表    |
| bit1  | A   | 非对齐 abort（boot 时清 0 → 允许非对齐）         |
| bit2  | C   | D-cache（C 可以没有 M 单独开：U-Boot 就这么活）  |
| bit12 | I   | I-cache                                         |
| bit22 | U   | v6 非对齐模型（ARM1176 需要）                    |
| bit28 | TRE | TEX 重映射（清 0 = 用经典 AP 编码）              |

### 4. 页表项格式（section, 32bit）

- `bit[1:0]` = `0b10`：section 类型
- `bit[31:20]` = `PA[31:20]`：物理 1MB 块号 ← 所以 `add #1048576` = 下 1MB
- `bit[11:10]` = `AP[2:1]=11`：全权限（`3<<10`）
- `bit[4:3]` = 0x18? 不，用贴纸：`0x04=B`、`0x02=C`、`0x10=XN/v4N`
- `bit31/v7 XN`：只拦取指，不拦数据 load ★

1:1 恒等映射的目的：不是翻译，是合法持有 C 位 + 解除 XN。

### 5. 点火仪式（__armv7_mmu_cache_on 流程）

1. `mrc ID_MMFR0` → 测 VMSA → 决定 r6 贴纸 & 是否 `__setup_mmu`
2. drain write buffer + flush TLB（开表前洗干净的旧账）
3. 攒 SCTLR 值到 r0：`I|C|W|A|U`...，唯独先不置 M
4. `mcr TTBR0(r3)` → DACR → TTBCR：硬件先就位
5. `mcr ISB`：冲掉预取的老指令
6. `mcr c1 (M=1)` ★ 点火：PC 是物理值 → 必须 1:1 才回得来
7. `mrc c1` 读回 + `mcr ISB`：强制写完成 + 二次消毒
8. `mov pc, r12`：返回

老核替身 `__common_mmu_cache_on`（无 ISB 时代）：

```asm
b 1f; .align 5; 1: mcr c1; mrc c1; sub pc, lr, r0, lsr #32
（lsr#32 恒 0 → 借一条无害指令冲刷流水线）
```

DEBUG 模式故意不 `orr #0x0d` → 不开 MMU，保串口。

### 6. 记不住怎么办

- 编码是查表件：`grep "c7, c10" arch/arm/include/asm/`，注释自带人话
- 只背三样：柜号地图 c1/c2/c3/c7、SCTLR bit0/2/12、点火顺序（表 → 域 → ISB → M → ISB）
- 其余信任 head.S 自己的注释 —— 作者也是抄 TRM 的

### 7. 场景速判

| 场景                              | 走法                                                 |
|-----------------------------------|------------------------------------------------------|
| U-Boot 已开 cache + zImage 在 RAM | `cache_on` 检查直接 skip（你的板子）                  |
| MMU 全关冷启动 + RAM              | 走全套，买的是 RAM 窗口的 C 位                       |
| XIP from NOR（ZBOOT_ROM）         | 全套 + 尾部 2MB 补丁（给 PC 取指，piggy 超出只是慢） |

XIP 省的是"压缩副本暂存 RAM"，解压输出那份 RAM 永远省不掉。

---

## 三、__setup_mmu 详解笔记（续上表，head.S:794-833）

### 1. 任务定位

纯软件函数（无 CP15），产出物 = RAM 里一本 4096 项页表。"设 MMU"是错觉：写表只是印字典；查字典是后面 `mcr c1` 点火的事。

- 写表（建数据） vs 启用（查表硬件）—— 两步永远分开。

### 2. 开局三件套

```asm
sub  r3, r4, #16384        @ 表放在内核落点前面 16K
bic  r3, #0xff / #0x3f00   @ 16K 对齐（编码限制拆两刀：0x3fff 不是合法立即数）
mov  r0, r3                @ r0 = 写游标
mov  r9, r0, lsr#18, lsl#18  @ RAM 窗口起点 = 表基址向下 1MB 对齐
add  r10, r9, #0x10000000    @ 窗口 = 256MB（r9 起向上）
```

### 3. 主循环：铺满 4GB

```asm
mov  r1, #0x12 | 3<<10   @ 起点: section(0b10) + XN|U + AP=11
1:  cmp   r1, r9         @ ┐ 复合条件 if (r9 <= virt < r10)
    cmphs r10, r1        @ ┘ 短路链: cmp→cmphs→后面的 orrhs/orrlo 吃 flags
    bic   r1, #0x1c      @ 洗掉上轮贴纸 XN|C|B
    orrlo r1, #0x10      @ 窗口外: XN（lo = cmp 判负 = virt < r9 或 r10 <= virt）
    orrhs r1, r6         @ 窗口内: 贴 caller 的 RAM 贴纸 r6
    str   r1, [r0], #4   @ ★ 这一格 = 当前 virt 对应的表项; 游标+4
    add   r1, #1048576   @ virt+1MB（PA 字段在 bit31:20，加1MB=加1个块号，低20位 flags 纹丝不动 —— 表项值就是虚拟地址本身）
    teq   r0, r2         @ 游标==表尾 r3+16384（4096 格×4B=4GB 全覆盖）
    bne   1b
```

**要点：**

- 1:1 → 第 n 格代表虚拟 n MB，值里 PA 也是 n MB；
- 4GB 不是映射 4GB 内存，是每个 1MB 都有通行证，缺格 = translation fault；
- 全铺的"其他格子"（UART 等外设）默认 XN+uncached = 恰好是外设正确属性；
- r6 哪来的：`__armv7` 分支 `movne r6, #CB_BITS|0x02`（VMSA 检测后塞的）；老核 r6 = `CB_BITS|0x12`。

### 4. 尾部补丁：给 PC 自己贴 2MB

```asm
orr  r1, r6, #0x04       @ 重拿贴纸+强制B（r1 循环完已绕回，不续用）
orr  r1, #3<<10          @ AP=11
mov  r2, pc              @ ★ flash 地址的唯一来源 = 运行时 PC 自己
mov  r2, r2, lsr #20     @ r2 = 我在第几个 1MB section（索引）
orr  r1, r1, r2, lsl #20 @ 把 PA 摆进描述符 bit31:20
add  r0, r3, r2, lsl #2  @ 槽位 = 表基址 + 索引×4
str  r1, [r0], #4        @ 盖掉主循环刚写的 XN 版（第 1MB）
add  r1, #1048576; str   @ 相邻下一格（第 2MB，防跨 1MB 边界）
mov  pc, lr              @ = C 的 return，与映射无关
```

两格 = 两条 1MB 映射，`str` 落下之前映射并不存在（不是"写回"）。

### 5. 补丁给谁用（易混点总清算）

- 用户只有两个：① XIP 时壳自己的取指 ② 解压时 `ldr` 读 flash 里的 piggy
- XN 只拦取指 → 代码必须全在 2MB 内（壳 .text 才几十 KB，够）
- 数据读到范围外 → 只是 uncached 慢，不 abort → 20MB 镜像也炸不了
- 栈/页表/heap/BSS/内核落点 全在 r4 附近 → 主循环 RAM 窗口的管辖内，轮不到补丁管
- 内核解压完直接在 RAM 中运行，跳 r4 后整本页表（含补丁）作废：内核自己的 head.S 会 cache_off 重建 —— 用完即焚
- 中断：此阶段 I/F 屏蔽、没设 V=1，全程无中断参与
- RAM 启动（U-Boot 已拷 0x80800000）：PC 也在 RAM 窗口里 → 补丁 = 重复劳动，白写两格（无害）；U-Boot 常已开 cache → 连 `__setup_mmu` 都被 cache_on 检查 skip

### 6. 为什么非得有表才能开 cache

ARM 上"这次访问走不走 cache"的属性只有两个来源：

- MMU 开 → 查描述符 C/B 位（细粒度，标准路）
- MMU 关 → PRRR/NMRR（c10）老路，粗且各家核不一

所以 `__setup_mmu` 买的是 C 位，翻译是搭车的恒等函数。收益：uncached QSPI 数十 MB/s ↔ cached L1 —— 解压流的输入/输出速度差几十倍。

### 7. XIP 存在意义（小 RAM 账本）

| 路线   | 峰值占 RAM | 组成                        |
|--------|-----------|-----------------------------|
| 非 XIP | 3+8+~1MB  | 压缩副本 + 解压输出 + 运行时 |
| XIP    | 8+~1MB    | 解压输出 + 运行时           |

省的是"暂存"，不是"输出"；输出那份 RAM 永远省不掉。现代 distro 全走 RAM 路线，`ZBOOT_ROM` ≈ 博物馆展品，GOT 反向规则/不能 relocate/2MB 补丁 都是它的配套遗产。
