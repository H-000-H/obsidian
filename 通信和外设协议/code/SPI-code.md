---
title: SPI 驱动实现完整手册（Bit-Bang / 轮询 / 中断 / DMA）
tags:
  - Embedded
  - Protocol
  - SPI
  - Driver
  - DMA
  - BitBanging
  - Python
  - C
status: Completed
created: 2026-09-20
---

# 🔌 SPI 驱动实现完整手册（Bit-Bang / 轮询 / 中断 / DMA）

> [!INFO] 笔记概览
> - **核心线索**：SPI 实现本质与 UART 一样，区分"谁来产生时钟/时序"与"谁来搬运数据"，据此分为四种方式。
> - **内容范围**：GPIO 软件模拟（Python）· 硬件轮询 · 硬件中断状态机 · SPI+DMA 双向搬运 · 四种方式对比 · CPU 参与度 · 抽象层级
> - **推荐路径**：§0 总览 → §1 Bit-Bang → §2 轮询 → §3 中断 → §4 DMA → §5 对比 → §8 总结
> - **代码约定**：Python 用于 Bit-Bang 逻辑模拟（PEP8）；C 用于硬件外设驱动（**Allman 大括号**、4 空格缩进、`snake_case`），均为教学用伪寄存器。

---

## 🗺️ 0. 四种实现方式总览

```text
软件模拟 SPI ──► GPIO 手工翻转 SCK / MOSI、读 MISO
硬件轮询 SPI ──► CPU 查询 SPI 状态寄存器（TXE / RXNE）
硬件中断 SPI ──► SPI 事件触发 IRQ ──► ISR 处理收发
SPI + DMA   ──► SPI 产生 DMA 请求 ──► DMA 双向自动搬运 ──► 完成后中断通知 CPU
```

> [!NOTE] SPI 的特殊点
> 因为**同步 + 主机提供时钟**，用软件模拟（主机侧）相对可靠；但 MISO 的采样时机必须对准 CPOL/CPHA 指定的边沿，这是 Bit-Bang 最容易出错的地方。

---

## 🖐️ 1. 软件模拟 SPI（GPIO Bit-Banging）

不使用 MCU 的 SPI 外设，由 CPU 用 GPIO 手动画出 SCK，并按模式确定采样边沿。先约定实现 **Mode 0（CPOL=0, CPHA=0）**：SCK 空闲为低、**上升沿采样**、默认 MSB 先行。

抽象 GPIO 接口（Python 模拟）：

```python
def gpio_write_sclk(level: int):
    print("SCLK=", level, sep="")

def gpio_write_mosi(level: int):
    print("MOSI=", level, sep="")

def gpio_write_cs(level: int):
    print("CS=", level, sep="")

def gpio_read_miso() -> int:
    return 1
```

### 1.1 Mode 0 主机收发一个字节（MSB 先行）

```python
def spi_xfer_byte(data: int, num_bits: int = 8):
    result = 0
    gpio_write_cs(0)                 # CS 拉低选通从机

    for i in range(num_bits):
        bit = (data >> (num_bits - 1 - i)) & 0x01   # MSB 先行

        gpio_write_sclk(0)           # SCK 低电平期间准备 MOSI
        gpio_write_mosi(bit)

        gpio_write_sclk(1)           # 上升沿：主机采样 MISO
        m = gpio_read_miso()
        result = (result << 1) | m

    gpio_write_sclk(0)               # 回到空闲电平
    gpio_write_cs(1)                 # CS 拉高释放总线
    return result
```

发送 `0x53`（MSB 先行）的位序：

```text
data  = 0x53 = 01010011b
i:     0     1     2     3     4     5     6     7
MOSI:  0     1     0     1     0     0     1     1
采样:  上升沿 上升沿 上升沿 上升沿 上升沿 上升沿 上升沿 上升沿
```

### 1.2 改变采样沿（适用于 Mode 1/2/3）

Mode 1（CPOL=0, CPHA=1）在**下降沿**采样，时序改为：先拉高 SCK 制造下降沿前提 → 下降沿采样 MISO → SCK=0 → 准备下一位 MOSI。

```python
def spi_xfer_byte_mode1(data: int, num_bits: int = 8):
    result = 0
    gpio_write_cs(0)

    for i in range(num_bits):
        bit = (data >> (num_bits - 1 - i)) & 0x01

        gpio_write_sclk(1)           # SCK 高电平期间准备数据
        gpio_write_mosi(bit)

        gpio_write_sclk(0)           # 下降沿：采样 MISO
        m = gpio_read_miso()
        result = (result << 1) | m

    gpio_write_cs(1)
    return result
```

> [!TIP] 位序切换
> 位序由 `num_bits - 1 - i` 控制（MSB 先行）。若需 LSB 先行，把索引改成 `i` 即可。模式与采样沿推导见 [[SPI]] §2。

---

## 🔁 2. 硬件 SPI：轮询方式

MCU 内部已有 SPI 外设，CPU 通过查询状态寄存器完成收发。**全双工下，发一个字节和读一个字节是同一个操作**：写一次 DR 发出 TX，同时从 DR 读回 RX。

```text
CPU ──寄存器──► SPI Peripheral ──┬── SCLK
                                 ├── MOSI
                                 ├── MISO
                                 └── CS
```

伪寄存器定义（C，Allman 风格）：

```c
#include <stdint.h>

typedef struct
{
    volatile uint32_t CR;   /* Control  */
    volatile uint32_t SR;   /* Status   */
    volatile uint32_t DR;   /* Data     */
} spi_regs_t;

#define SPI0 ((spi_regs_t *)0x40020000UL)

#define SPI_SR_TXE  (1U << 0)   /* TX 空，可写数据 */
#define SPI_SR_RXNE (1U << 1)   /* RX 非空，可读数据 */

#define SPI_CR_SPE    (1U << 0)  /* SPI 使能 */
#define SPI_CR_MSTR   (1U << 1)  /* 主机模式 */
#define SPI_CR_MODE0  (0U << 2)  /* Mode 0：CPOL=0, CPHA=0 */
```

读写一个字节：

```c
uint8_t spi_xfer(uint8_t data)
{
    /* 等待 TX 空 */
    while ((SPI0->SR & SPI_SR_TXE) == 0)
    {
    }

    /* 写入=发出，同时启动接收 */
    SPI0->DR = data;

    /* 等待 RX 非空 */
    while ((SPI0->SR & SPI_SR_RXNE) == 0)
    {
    }

    /* 读回从机数据 */
    return (uint8_t)SPI0->DR;
}
```

连续读写：

```c
void spi_write_read(const uint8_t *tx, uint8_t *rx, uint32_t len)
{
    for (uint32_t i = 0; i < len; i++)
    {
        rx[i] = spi_xfer(tx[i]);
    }
}
```

> [!WARNING] 轮询的缺点
> 与 UART 相同：`while(...)` 自旋消耗大量 CPU，无法在等待期间做其它事。

---

## 🔔 3. 硬件 SPI：中断方式

中断方式让 CPU 不必持续自旋，用环形缓冲管理收发。补充寄存器位与缓冲区：

```c
#define SPI_CR_TXIE  (1U << 3)  /* 使能 TX 中断 */
#define SPI_CR_RXIE  (1U << 4)  /* 使能 RX 中断 */

#define SPI_TX_BUF_SIZE 128
#define SPI_RX_BUF_SIZE 128

static volatile uint8_t  tx_buf[SPI_TX_BUF_SIZE];
static volatile uint8_t  rx_buf[SPI_RX_BUF_SIZE];
static volatile uint32_t tx_head, tx_tail;
static volatile uint32_t rx_head, rx_tail;
```

### 3.1 应用层提交发送

```c
int spi_write_byte(uint8_t data)
{
    uint32_t next = (tx_head + 1) % SPI_TX_BUF_SIZE;
    if (next == tx_tail)
    {
        return 0;   /* 缓冲区满 */
    }
    tx_buf[tx_head] = data;
    tx_head = next;

    SPI0->CR |= SPI_CR_TXIE;   /* 开启 TX 中断 */
    return 1;
}
```

### 3.2 SPI ISR（收发共用一个中断向量的典型外设）

```c
void SPI0_IRQHandler(void)
{
    uint32_t sr = SPI0->SR;

    /* RX 非空：读回从机返回的数据 */
    if (sr & SPI_SR_RXNE)
    {
        uint8_t d = (uint8_t)SPI0->DR;

        uint32_t next = (rx_head + 1) % SPI_RX_BUF_SIZE;
        if (next != rx_tail)
        {
            rx_buf[rx_head] = d;
            rx_head = next;
        }
    }

    /* TX 空且有待发数据 */
    if (sr & SPI_SR_TXE)
    {
        if (tx_head != tx_tail)
        {
            SPI0->DR = tx_buf[tx_tail];
            tx_tail = (tx_tail + 1) % SPI_TX_BUF_SIZE;
        }
        else
        {
            /* 无数据时关闭 TX 中断，避免 IRQ 风暴 */
            SPI0->CR &= ~SPI_CR_TXIE;
        }
    }
}
```

> [!CAUTION] 全双工注意点
> 从机只有在被 CS 选中且主机持续给出 SCK 时才回数据。若一次传输 RX 全为 `0xFF`，通常是 CS 未选中或主机没有提供足够时钟周期。环形缓冲与 ISR 下半部处理见 [[DMA、Buffer 与流量控制]]。

---

## 🚀 4. 硬件 SPI + DMA

DMA 的作用仍是**搬运**，而不是让时钟更快：

```text
Memory ──► TX DMA ──► SPI DR ──► SPI MOSI ──► (从机)
Memory ◄── RX DMA ◄── SPI DR ◄── SPI MISO ◄── (从机)
```

DMA 伪寄存器（补齐源/目的地址模式宏）：

```c
typedef struct
{
    volatile uint32_t SRC;
    volatile uint32_t DST;
    volatile uint32_t LEN;
    volatile uint32_t CTRL;
    volatile uint32_t STATUS;
} dma_regs_t;

#define DMA0 ((dma_regs_t *)0x50000000UL)

#define DMA_CTRL_ENABLE     (1U << 0)   /* 通道使能 */
#define DMA_CTRL_SRC_INC    (1U << 1)   /* 源地址递增 */
#define DMA_CTRL_SRC_FIXED  (1U << 2)   /* 源地址固定 */
#define DMA_CTRL_DST_INC    (1U << 3)   /* 目的地址递增 */
#define DMA_CTRL_DST_FIXED  (1U << 4)   /* 目的地址固定 */
#define DMA_CTRL_TXIE       (1U << 5)   /* 传输完成中断使能 */

#define DMA_STATUS_DONE     (1U << 0)
```

### 4.1 SPI TX DMA（内存 → SPI DR）

```c
void spi_dma_write(const uint8_t *buf, uint32_t len)
{
    DMA0->SRC = (uint32_t)buf;          /* 源=内存（递增） */
    DMA0->DST = (uint32_t)&SPI0->DR;    /* 目的=SPI DR（固定） */
    DMA0->LEN = len;

    DMA0->CTRL =
        DMA_CTRL_SRC_INC |
        DMA_CTRL_DST_FIXED |
        DMA_CTRL_TXIE |
        DMA_CTRL_ENABLE;

    SPI0->CR |= (1U << 6);              /* SPI 打开 TX DMA 请求 */
}
```

### 4.2 SPI RX DMA（SPI DR → 内存）

```c
static uint8_t rx_dma_buf[128];

void spi_dma_rx_start(void)
{
    DMA0->SRC = (uint32_t)&SPI0->DR;    /* 源=SPI DR（固定） */
    DMA0->DST = (uint32_t)rx_dma_buf;   /* 目的=内存（递增） */
    DMA0->LEN = sizeof(rx_dma_buf);

    DMA0->CTRL =
        DMA_CTRL_DST_INC |
        DMA_CTRL_SRC_FIXED |
        DMA_CTRL_TXIE |
        DMA_CTRL_ENABLE;

    SPI0->CR |= (1U << 7);              /* SPI RX DMA 请求 */
}
```

### 4.3 DMA 完成中断

```c
static volatile int spi_dma_busy;

void DMA0_IRQHandler(void)
{
    if (DMA0->STATUS & DMA_STATUS_DONE)
    {
        DMA0->STATUS = DMA_STATUS_DONE;   /* 清标志 */
        DMA0->CTRL &= ~DMA_CTRL_ENABLE;
        spi_dma_busy = 0;
    }
}
```

> [!WARNING] DMA 三大常见坑
> 1. **DMA 完成 ≠ 传输完成**：DMA 把数据搬进 SPI DR 后，移位寄存器可能仍在串行输出。真正"发完"需等 SPI Busy 标志清零。
> 2. **RX 必须与 TX 同时跑**：全双工下不提供时钟就没数据。若只配 RX DMA，需额外"喂"驱动时钟（写 `0xFF` 填充）。
> 3. **CS 的软件时序**：很多 MCU 的 CS 不由 SPI+DMA 自动管理，需在启动 DMA 前手动拉低、结束后拉高。
>
> 高速 SPI + DMA 还须处理 Cache 一致性，见 [[SPI-DMA-多核-Cache]]。

---

## 🆚 5. 四种方式对比

| 方式 | CPU 占用 | 实现复杂度 | 数据量 | 实时性 | 典型用途 |
| :--- | :--: | :--: | :--: | :--: | :--- |
| **软件模拟 SPI** | 很高 | 中 | 很低 | 较差 | 无 SPI 外设、特殊 GPIO 复用、DEBUG |
| **硬件轮询** | 高 | 最低 | 低 | 简单直接 | 少量寄存器读写、初始化 |
| **硬件中断** | 中低 | 中 | 中 | 高 | 普通外设驱动、传感器 |
| **SPI + DMA** | 最低 | 高 | 大 | 高 | LCD 刷屏、SD 卡、大批量数据搬运 |

---

## 🧮 6. CPU 参与程度

| 方式 | 每单位操作 | CPU 状态 |
| :--- | :--- | :--- |
| 软件模拟 SPI | 每一拍都要翻转 SCK 和 MOSI（1 字节 ≥ 8 次采样操作） | 全程占用 |
| 硬件轮询 | 每字节忙等 TXE / RXNE | 只在等待，未释放 |
| 硬件中断 | 每字节 / 每 FIFO 事件进入 ISR | 事件驱动 |
| DMA | 整个 Buffer 只配置一次 + 等完成 | 几乎释放 |

---

## 🏗️ 7. 抽象层级

```text
软件模拟 SPI:   Application → GPIO 位移+翻转 SCK → Pin
硬件 SPI:       Application → SPI Driver → SPI Peripheral → Pin
硬件 SPI + DMA: Application → Buffer ⇄ DMA ⇄ SPI Peripheral
```

---

## 🧠 8. 总结

SPI 实现本质与 UART 一样，区分"谁来产生时钟/时序"与"谁来搬运数据"：

```text
Bit-Bang：  时序=CPU   搬数据=CPU
Polling：   时序=SPI   搬数据=CPU 等待
IRQ：       时序=SPI   搬数据=CPU 被通知
DMA：       时序=SPI   搬数据=DMA / CPU 只做配置与收尾
```

> [!SUCCESS] 与 UART 最大的不同
> SPI 全双工、无帧起始/停止额外开销、速率可达 MHz 级；对应地，实现时必须先确认主从双方在 **CPOL/CPHA、位序（MSB 先行为主）、字长、CS 极性**上完全对齐，否则可能"通信逻辑正确却数据错位"。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] SPI 驱动上线前必过
> - [ ] **模式对齐**：CPOL/CPHA 与从机一致，Bit-Bang 采样沿对准
> - [ ] **位序对齐**：确认 MSB / LSB 先行，字长 8/16/32 一致
> - [ ] **方式选型**：少量读写用轮询，中等用中断，大批量用 DMA
> - [ ] **全双工认知**：写 DR 即同时收发；纯读需喂 dummy 提供时钟
> - [ ] **中断缓冲**：环形缓冲 head/tail 做临界区保护，防空转 IRQ 风暴
> - [ ] **DMA 宏完整**：源/目的地址递增与固定模式宏都定义，避免未定义引用
> - [ ] **DMA 完成语义**：DMA TCIF 后仍需等 SPI Busy 清零才算物理发完
> - [ ] **CS 时序**：DMA 前手动拉低 CS、结束后拉高（若非硬件自动管理）
> - [ ] **Cache 一致性**：高速 DMA 发送前 Clean、接收后 Invalidate

---

## 🔗 思考与延伸索引

- [[SPI]] — SPI 协议层总览：CPOL/CPHA、帧格式、多线 SPI
- [[SPI-DMA-多核-Cache]] — SPI + DMA 高速链路的 Cache 一致性与多核交接
- [[DMA、Buffer 与流量控制]] — 环形缓冲、DMA TC 语义与背压
- [[UART-code]] — 与 SPI 实现方式高度相似的 UART 驱动四态
- [[I2C-code]] — 另一种同步总线的驱动实现对照

---

## 📚 参考资料

1. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（SPI / DMA 章节）.
2. **STMicroelectronics** — *AN2811: SPI hardware emulation using GPIO*.
3. **Microchip** — *Application Note: SPI Master/Slave Bit-Banging Techniques*.
4. 项目内部笔记：[[SPI]]、[[SPI-DMA-多核-Cache]]。
