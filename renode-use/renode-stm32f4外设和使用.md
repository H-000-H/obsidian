---
title: Renode STM32F4 平台外设清单与地址映射手册
tags:
  - Renode
  - STM32F4
  - Simulation
  - Peripheral
  - Memory-Map
  - Embedded
status: Completed
---

# 🧩 Renode STM32F4 平台外设清单与地址映射手册

> [!INFO] 笔记概览
> - **核心命题**：Renode 用 `.repl` 把 STM32F4 Discovery 的外设、地址、总线一次性建出来；本手册是 `peripherals` 命令输出的**分类速查表**，用于快速定位模型名、地址范围与可用外设。
> - **内容范围**：如何查看外设树 · 存储/内核/DMA/通信/定时器/GPIO 分类表 · 关键地址速查 · 完整外设树原始输出
> - **推荐路径**：§1 查看方法 → §2~§7 分类速查 → §8 完整树 → §9 引用外设
> - **平台基准**：以 `platforms/boards/stm32f4_discovery.repl`（STM32F407）为例，其它 STM32F4 板大同小异。

---

## 🔍 1. 如何查看外设树

在已加载平台的机器上下文里执行 `peripherals`，即可得到带地址范围的外设树：

```text
peripherals                              # 列出全部外设（带地址范围树）
peripherals uart                         # 按名字模糊过滤
peripherals STM32_Timer                  # 按模型名过滤
peripherals <0x40000000, 0x7FFFFFFF>     # 按地址范围过滤
```

加载平台：

```text
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery.repl
```

> [!TIP] 输出即文档
> `peripherals` 的树形输出同时给出**模型名**（如 `STMCAN`）与**地址范围**，是写 `.repl`、配 DMA、核对 `compatible` 时最权威的一手资料。

---

## 💾 2. 存储与位带（Memory / BitBanding）

| 外设名 | 模型 | 地址范围 | 说明 |
| :--- | :--- | :--- | :--- |
| `flash` | MappedMemory | `0x08000000–0x081FFFFF` | 主 Flash（2MB） |
| `sram` | MappedMemory | `0x20000000–0x2003FFFF` | 主 SRAM（256KB） |
| `rom1` | MappedMemory | `0x1FFF0000–0x1FFFBFFF` | 系统 ROM |
| `rom2` | MappedMemory | `0x1FFFC400–0x1FFFFFFF` | 系统 ROM（续） |
| `fsmcBank1` | MappedMemory | `0x60000000–0x6FFFFFFF` | FSMC 外部存储 Bank1 |
| `bitbandPeripherals` | BitBanding | `0x42000000–0x43FFFFFF` | 外设位带别名区 |
| `bitbandSram` | BitBanding | `0x22000000–0x221FFFFF` | SRAM 位带别名区 |

---

## ⚙️ 3. 内核与系统（Core / System）

| 外设名 | 模型 | 地址范围 | 说明 |
| :--- | :--- | :--- | :--- |
| `cpu` | CortexM | Slot: 0 | Cortex-M4 内核 |
| `nvic` | NVIC | `0xE000E000–0xE000EFFF` | 中断控制器 |
| `rcc` | STM32F4_RCC | `0x40023800–0x40023BFF` | 复位与时钟控制 |
| `pwr` | STM32_PWR | `0x40007000–0x400073FF` | 电源控制 |
| `flash_controller` | STM32F4_FlashController | `0x40023C00–0x40023FFF` | Flash 控制器 |
| ↳ optionBytes | — | `0x1FFFC000–0x1FFFC0FF` | 选项字节区 |
| `crc` | STM32_CRC | `0x40023000–0x400233FF` | 硬件 CRC |
| `rng` | STM32F4_RNG | `0x50060800–0x50060BFF` | 随机数发生器 |
| `rtc` | STM32F4_RTC | `0x40002800–0x40002BFF` | 实时时钟 |
| `iwdg` | STM32_IndependentWatchdog | `0x40003000–0x400033FF` | 独立看门狗 |
| `exti` | STM32F4_EXTI | `0x40013C00–0x40013FFF` | 外部中断控制器 |

---

## 🚚 4. DMA 控制器

| 外设名 | 模型 | 地址范围 |
| :--- | :--- | :--- |
| `dma1` | STM32DMA | `0x40026000–0x400263FF` |
| `dma2` | STM32DMA | `0x40026400–0x400267FF` |

> [!NOTE] DMA 与外设联动
> DMA 搬运的源/目的地址需与外设 DATA 寄存器对齐。DMA 机制与 TC 语义见 [[DMA、Buffer 与流量控制]]，驱动实现见 [[UART-code]]、[[SPI-code]]。

---

## 📡 5. 通信外设（UART / I²C / SPI / CAN / ETH）

| 外设名 | 模型 | 地址范围 |
| :--- | :--- | :--- |
| `usart1` | STM32_UART | `0x40011000–0x400110FF` |
| `usart2` | STM32_UART | `0x40004400–0x400044FF` |
| `usart3` | STM32_UART | `0x40004800–0x400048FF` |
| `uart4` | STM32_UART | `0x40004C00–0x40004CFF` |
| `uart5` | STM32_UART | `0x40005000–0x400050FF` |
| `i2c1` | STM32F4_I2C | `0x40005400–0x400057FF` |
| `i2c2` | STM32F4_I2C | `0x40005800–0x40005BFF` |
| `i2c3` | STM32F4_I2C | `0x40005C00–0x40005FFF` |
| `spi1` | STM32SPI | `0x40013000–0x400133FF` |
| `spi2` | STM32SPI | `0x40003800–0x40003BFF` |
| `spi3` | STM32SPI | `0x40003C00–0x40003FFF` |
| `can1` | STMCAN | `0x40006400–0x400067FF` |
| `can2` | STMCAN | `0x40006800–0x40006BFF` |
| `ethernet` | SynopsysEthernetMAC | `0x40028000–0x400293FF` |

> [!TIP] 串口仿真对接
> 拿到 `usart2` 后，用 `showAnalyzer sysbus.usart2` 弹窗，或 `emulation CreateServerSocketTerminal 1234 "term"` + `connector Connect` 接到 TCP。详见 [[renode外设仿真]] §1。

---

## ⏱️ 6. 定时器（Timers）

| 外设名 | 模型 | 地址范围 | 外设名 | 模型 | 地址范围 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `timer1` | STM32_Timer | `0x40010000–0x400103FF` | `timer8` | STM32_Timer | `0x40010400–0x400107FF` |
| `timer2` | STM32_Timer | `0x40000000–0x400003FF` | `timer9` | STM32_Timer | `0x40014000–0x400143FF` |
| `timer3` | STM32_Timer | `0x40000400–0x400007FF` | `timer10` | STM32_Timer | `0x40014400–0x400147FF` |
| `timer4` | STM32_Timer | `0x40000800–0x40000BFF` | `timer11` | STM32_Timer | `0x40014800–0x40014BFF` |
| `timer5` | STM32_Timer | `0x40000C00–0x40000FFF` | `timer12` | STM32_Timer | `0x40001800–0x40001BFF` |
| `timer6` | STM32_Timer | `0x40001000–0x400013FF` | `timer13` | STM32_Timer | `0x40001C00–0x40001FFF` |
| `timer7` | STM32_Timer | `0x40001400–0x400017FF` | `timer14` | STM32_Timer | `0x40002000–0x400023FF` |

---

## 🔘 7. GPIO 端口（含板载 LED / 按键）

| 外设名 | 模型 | 地址范围 | 挂载子外设 |
| :--- | :--- | :--- | :--- |
| `gpioPortA` | STM32_GPIOPort | `0x40020000–0x400203FF` | └── `UserButton`（Button） |
| `gpioPortB` | STM32_GPIOPort | `0x40020400–0x400207FF` | — |
| `gpioPortC` | STM32_GPIOPort | `0x40020800–0x40020BFF` | — |
| `gpioPortD` | STM32_GPIOPort | `0x40020C00–0x40020FFF` | └── `UserLED`（LED） |
| `gpioPortE` | STM32_GPIOPort | `0x40021000–0x400213FF` | — |
| `gpioPortF` | STM32_GPIOPort | `0x40021400–0x400217FF` | — |
| `gpioPortG` | STM32_GPIOPort | `0x40021800–0x40021BFF` | — |
| `gpioPortH` | STM32_GPIOPort | `0x40021C00–0x40021FFF` | — |
| `gpioPortI` | STM32_GPIOPort | `0x40022000–0x400223FF` | — |
| `gpioPortJ` | STM32_GPIOPort | `0x40022400–0x400227FF` | — |
| `gpioPortK` | STM32_GPIOPort | `0x40022800–0x40022BFF` | — |

> [!NOTE] 板载外设
> Discovery 板的 `UserButton`（PA0）与 `UserLED`（PD12–PD15）已作为 GPIO 子外设建好，可直接 `sysbus.gpioPortA.UserButton Press` 模拟按下。LED / 按键仿真见 [[renode外设仿真]] §2。

---

## 🌳 8. 完整外设树（`peripherals` 原始输出）

```text
(stm32f407_platform) peripherals
Available peripherals:
  sysbus (SystemBus)
  ├── bitbandPeripherals (BitBanding)   <0x42000000, 0x43FFFFFF>
  ├── bitbandSram (BitBanding)          <0x22000000, 0x221FFFFF>
  ├── can1 (STMCAN)                     <0x40006400, 0x400067FF>
  ├── can2 (STMCAN)                     <0x40006800, 0x40006BFF>
  ├── cpu (CortexM)                     Slot: 0
  ├── crc (STM32_CRC)                   <0x40023000, 0x400233FF>
  ├── dma1 (STM32DMA)                   <0x40026000, 0x400263FF>
  ├── dma2 (STM32DMA)                   <0x40026400, 0x400267FF>
  ├── ethernet (SynopsysEthernetMAC)    <0x40028000, 0x400293FF>
  ├── exti (STM32F4_EXTI)               <0x40013C00, 0x40013FFF>
  ├── flash (MappedMemory)              <0x08000000, 0x081FFFFF>
  ├── flash_controller (STM32F4_FlashController)
  │                                     <0x40023C00, 0x40023FFF>
  │                                     <0x1FFFC000, 0x1FFFC0FF> [region: optionBytes]
  ├── fsmcBank1 (MappedMemory)          <0x60000000, 0x6FFFFFFF>
  ├── gpioPortA (STM32_GPIOPort)        <0x40020000, 0x400203FF>
  │   └── UserButton (Button)
  ├── gpioPortB (STM32_GPIOPort)        <0x40020400, 0x400207FF>
  ├── gpioPortC (STM32_GPIOPort)        <0x40020800, 0x40020BFF>
  ├── gpioPortD (STM32_GPIOPort)        <0x40020C00, 0x40020FFF>
  │   └── UserLED (LED)
  ├── gpioPortE (STM32_GPIOPort)        <0x40021000, 0x400213FF>
  ├── gpioPortF (STM32_GPIOPort)        <0x40021400, 0x400217FF>
  ├── gpioPortG (STM32_GPIOPort)        <0x40021800, 0x40021BFF>
  ├── gpioPortH (STM32_GPIOPort)        <0x40021C00, 0x40021FFF>
  ├── gpioPortI (STM32_GPIOPort)        <0x40022000, 0x400223FF>
  ├── gpioPortJ (STM32_GPIOPort)        <0x40022400, 0x400227FF>
  ├── gpioPortK (STM32_GPIOPort)        <0x40022800, 0x40022BFF>
  ├── i2c1 (STM32F4_I2C)                <0x40005400, 0x400057FF>
  ├── i2c2 (STM32F4_I2C)                <0x40005800, 0x40005BFF>
  ├── i2c3 (STM32F4_I2C)                <0x40005C00, 0x40005FFF>
  ├── iwdg (STM32_IndependentWatchdog)  <0x40003000, 0x400033FF>
  ├── nvic (NVIC)                       <0xE000E000, 0xE000EFFF>
  ├── pwr (STM32_PWR)                   <0x40007000, 0x400073FF>
  ├── rcc (STM32F4_RCC)                 <0x40023800, 0x40023BFF>
  ├── rng (STM32F4_RNG)                 <0x50060800, 0x50060BFF>
  ├── rom1 (MappedMemory)               <0x1FFF0000, 0x1FFFBFFF>
  ├── rom2 (MappedMemory)               <0x1FFFC400, 0x1FFFFFFF>
  ├── rtc (STM32F4_RTC)                 <0x40002800, 0x40002BFF>
  ├── spi1 (STM32SPI)                   <0x40013000, 0x400133FF>
  ├── spi2 (STM32SPI)                   <0x40003800, 0x40003BFF>
  ├── spi3 (STM32SPI)                   <0x40003C00, 0x40003FFF>
  ├── sram (MappedMemory)               <0x20000000, 0x2003FFFF>
  ├── timer1 (STM32_Timer)              <0x40010000, 0x400103FF>
  ├── timer2 (STM32_Timer)              <0x40000000, 0x400003FF>
  ├── timer3 (STM32_Timer)              <0x40000400, 0x400007FF>
  ├── timer4 (STM32_Timer)              <0x40000800, 0x40000BFF>
  ├── timer5 (STM32_Timer)              <0x40000C00, 0x40000FFF>
  ├── timer6 (STM32_Timer)              <0x40001000, 0x400013FF>
  ├── timer7 (STM32_Timer)              <0x40001400, 0x400017FF>
  ├── timer8 (STM32_Timer)              <0x40010400, 0x400107FF>
  ├── timer9 (STM32_Timer)              <0x40014000, 0x400143FF>
  ├── timer10 (STM32_Timer)             <0x40014400, 0x400147FF>
  ├── timer11 (STM32_Timer)             <0x40014800, 0x40014BFF>
  ├── timer12 (STM32_Timer)             <0x40001800, 0x40001BFF>
  ├── timer13 (STM32_Timer)             <0x40001C00, 0x40001FFF>
  ├── timer14 (STM32_Timer)             <0x40002000, 0x400023FF>
  ├── uart4 (STM32_UART)                <0x40004C00, 0x40004CFF>
  ├── uart5 (STM32_UART)                <0x40005000, 0x400050FF>
  ├── usart1 (STM32_UART)               <0x40011000, 0x400110FF>
  ├── usart2 (STM32_UART)               <0x40004400, 0x400044FF>
  └── usart3 (STM32_UART)               <0x40004800, 0x400048FF>
```

---

## 🎯 9. 引用外设：属性与行为

在机器上下文里用 `sysbus.<外设名>` 引用外设，可读属性、调行为：

```text
sysbus.sram Size                      # get 属性
sysbus.sram Size 0x20000              # set 属性
sysbus.sram ZeroAll                   # 调用行为（方法）
sysbus.usart2                         # 不带参数 = 列出该外设全部属性/方法及用法
```

> [!TIP] 忘记用法就问外设本身
> 输入 `sysbus.<外设名>` 不跟任何参数，会打印它全部可用的属性和方法，比翻文档更快。

---

## 🔗 思考与延伸索引

- [[renode用法]] — 机器生命周期、平台加载、固件加载与 `.resc` 脚本
- [[renode外设仿真]] — UART / GPIO / LED / 按键 / CAN 的仿真与可视化操作
- [[renode调试与仿真]] — GDB 调试、日志、快照与反向执行
- [[DMA、Buffer 与流量控制]] — DMA 外设的地址模式与搬运语义
- [[UART-code]] — 被仿真 UART 外设的驱动实现

---

## 📚 参考资料

1. **Renode 官方文档** — *Platform Description Files (.repl)* / *Basic Examples*（readthedocs）.
2. **Renode 仓库** — `platforms/boards/stm32f4_discovery.repl`、`platforms/cpus/stm32f4.repl`.
3. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（外设地址映射权威来源）.
4. 项目内部笔记：[[renode用法]]、[[renode外设仿真]]。
