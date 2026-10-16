---
title: ARM 汇编指令速查补充
tags:
  - linux-kernel
  - arm
  - assembly
  - cp15
status: Completed
---

# ⚙️ ARM 汇编指令速查补充

> [!INFO] 笔记概览
> - **定位**：ARM 启动路径与内核初始化中反复出现的汇编指令/伪指令速记（CP15 读写、位测试、移位、地址取址），是读懂 `head.S` 与启动内核的工具卡。
> - **内容范围**：CP15 指令 · 位测试/逻辑运算 · 分支 · 取址加载 · 移位
> - **推荐路径**：按指令类型分节速查，配合 `head.S` 实际代码回读。

---

## 🧭 1. CP15 协处理器读写

| 指令 | 语义 | 示例 |
| :---: | :--- | :--- |
| **mrc** | 从 CP15 读出到通用寄存器 | `mrc p15, 0, r0, c1, c0, 0` —— 读 SCTLR（MMU/Cache 开关）到 `r0` |
| **mcr** | 从通用寄存器写入 CP15 | `mcr p15, 0, r0, c1, c0, 0` —— 把修改好的 `r0` 写回 SCTLR |

> [!NOTE] 记忆方向
> `rd` 反着记：**`mrc` 是 "Move Register from Coprocessor to Register"**（协处理器 → 通用寄存器 读）；`mcr` 相反（通用寄存器 → 协处理器 写）。`p15` 表示 CP15。

---

## 🔍 2. 位测试与逻辑运算

### 2.1 TST：位测试

检查寄存器里**某一个 bit 有没有置 1**：

```asm
tst r0, #(1 << 5)        ; r0 & (1<<5)
```

- 与运算结果 ≠ 0 → `Z` 标志 = 0
- 与运算结果 = 0 → `Z` 标志 = 1

### 2.2 ORR：按位或

```asm
orr r0, r0, #(1 << 0)    ; r0 = r0 | 立即数
```

> [!TIP] TST + BNE 组合
> `bne label` 搭配 TST：**若检测的 bit 是 1（结果非零），就跳转到 label**。注意 **BNE = 不为零就跳；BEQ = 等于零才跳**，二者方向相反。

### 2.3 BIC：位清零

```asm
bic r0, r0, #(1 << 2)    ; r0 = r0 & ~立即数
```

---

## 🧭 3. 取址与加载

| 指令 | 语义 | 示例 |
| :---: | :--- | :--- |
| **adr** | 按 PC 计算相对偏移地址存到寄存器 | `adr r0, #x` → 把 x 相对当前指令的真实地址放入 `r0`（PIC 锚点，平移不变） |
| **ldrb** | Load Register Byte，从内存读 1 字节到寄存器低 8 位（高 24 位清零） | `ldrb \res, [\tmp1]` → 以 `\tmp1` 值为内存地址；执行后 `\res = *(uint8_t *)\tmp1` |

> [!WARNING] `adr` 与绝对加载的区别
> 在 PIC 位置无关代码中，`adr` 得到的是**运行时真实地址**（适合拿锚点再叠加差值）；如需链接期绝对地址则应配合 `ldr \tmp, [\res]` 读差值再 `add`，而不是直接依赖 `adr` 结果做跨段跳转。

---

## 🔄 4. 移位

| 操作 | 示例 | 语义 |
| :---: | :--- | :--- |
| 逻辑左移 **lsl** | `r0, lsl #8` | `r0` 左移 8 位 |
| 逻辑右移 **lsr** | `r0, lsr #8` | `r0` 右移 8 位 |

---

## 📚 参考资料

1. ARM Assembly Language — `adr`/`ldrb`/`tst` 指令语义（ARMv7-A/R ARM 手册）.
2. Linux — `arch/arm/boot/compressed/head.S`（`adr`/`ldr` 差值还原）、`arch/arm/kernel/head.S`（`mrc`/`mcr` 开关 SCTLR）.
