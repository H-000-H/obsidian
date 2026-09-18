# SPI 实现方式

SPI 常见软件实现方式与 UART 类似，可划分为：

1. 软件模拟 SPI（GPIO Bit-Banging，手动产生 SCK 与数据）
2. 硬件 SPI + 轮询
3. 硬件 SPI + 中断
4. 硬件 SPI + DMA

整体关系：

```text
软件模拟 SPI
    ↓
GPIO 手工翻转 SCK / MOSI、读 MISO

硬件轮询 SPI
    ↓
CPU 查询 SPI 状态寄存器（TXE / RXNE）

硬件中断 SPI
    ↓
SPI 事件触发 IRQ
    ↓
ISR 处理收发

SPI + DMA
    ↓
SPI 产生 DMA 请求
    ↓
DMA 自动搬运数据（TX 和 RX 双向）
    ↓
完成后中断通知 CPU
```

> SPI 的特殊点：因为**同步 + 主从由主机提供时钟**，用软件模拟（主机侧）相对可靠；而 MISO 的采样时机必须对准 CPOL/CPHA 指定的边沿，这是 Bit-Bang 最容易出错的地方。

---

# 1. 软件模拟 SPI（GPIO Bit-Banging）

不使用 MCU 的 SPI 外设，由 CPU 用 GPIO 手动画出 SCK，并按模式确定采样边沿。

先约定要实现的模式：**Mode 0（CPOL=0, CPHA=0）**，即：

- SCK 空闲为低
- 上升沿采样（主机在该沿读 MISO）
- 默认 MSB 先行

抽象 GPIO 接口：

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

## Mode 0 主机收发一个字节（MSB 先行）

```python
def spi_xfer_byte(data: int, num_bits: int = 8):
    result = 0
    # CS 拉低选通从机
    gpio_write_cs(0)

    for i in range(num_bits):
        # 取当前位：MSB 先行
        bit = (data >> (num_bits - 1 - i)) & 0x01

        # SCK 低电平期间把 MOSI 准备好
        gpio_write_sclk(0)
        gpio_write_mosi(bit)

        # 上升沿：主机采样 MISO
        gpio_write_sclk(1)
        m = gpio_read_miso()
        result = (result << 1) | m

    # 回到空闲电平
    gpio_write_sclk(0)

    # CS 拉高释放总线
    gpio_write_cs(1)

    return result
```

发送 0x53（MSB 先行）：

```text
data  = 0x53 = 01010011b
MSB→LSB:  0  1  0  1  0  0  1  1
```

每个循环一拍：

```text
i:     0     1     2     3     4     5     6     7
MOSI:  0     1     0     1     0     0     1     1
采样:  上升沿 上升沿 上升沿 上升沿 上升沿 上升沿 上升沿 上升沿
```

## 改变采样沿（适用于 Mode 1/2/3）

Mode 1（CPOL=0, CPHA=1）在**下降沿**采样，时序改成：

```text
SCK=1（先把时钟拉高，制造下降沿前提）
    ↓
下降沿：主机采样 MISO
    ↓
SCK=0
    ↓
MOSI 准备好下一位
```

```python
def spi_xfer_byte_mode1(data: int, num_bits: int = 8):
    result = 0
    gpio_write_cs(0)

    for i in range(num_bits):
        bit = (data >> (num_bits - 1 - i)) & 0x01

        # SCK 高电平期间把数据准备好
        gpio_write_sclk(1)
        gpio_write_mosi(bit)

        # 下降沿：采样 MISO
        gpio_write_sclk(0)
        m = gpio_read_miso()
        result = (result << 1) | m

    gpio_write_cs(1)
    return result
```

> 位序由 `num_bits - 1 - i` 控制：MSB 先行。若需要 LSB 先行，改成 `i` 即可。

---

# 2. 硬件 SPI：轮询方式

MCU 内部已有 SPI 外设：

```text
CPU
 │
 │ 寄存器
 ▼
SPI Peripheral
 │
 ├── SCLK
 ├── MOSI
 ├── MISO
 └── CS
```

CPU 通过查询 SPI 状态寄存器完成一次收发。全双工下，**发一个字节和读一个字节是同一个操作**：写一次 DR 发出 TX，同时从 DR 读回 RX。

伪寄存器定义：

```c
#include <stdint.h>

typedef struct {
    volatile uint32_t CR;   /* Control  */
    volatile uint32_t SR;   /* Status   */
    volatile uint32_t DR;   /* Data     */
} spi_regs_t;

#define SPI0 ((spi_regs_t *)0x40020000UL)

#define SPI_SR_TXE  (1U << 0)   /* TX 空，可写数据 */
#define SPI_SR_RXNE (1U << 1)   /* RX 非空，可读数据 */

#define SPI_CR_SPE   (1U << 0)  /* SPI 使能 */
#define SPI_CR_MSTR  (1U << 1)  /* 主机模式 */
#define SPI_CR_MODE0 (0U << 2)  /* Mode 0：CPOL=0, CPHA=0 */
```

读写一个字节：

```c
uint8_t spi_xfer(uint8_t data)
{
    /* 等待 TX 空 */
    while ((SPI0->SR & SPI_SR_TXE) == 0) {
    }

    /* 写入=发出，同时启动接收 */
    SPI0->DR = data;

    /* 等待 RX 非空 */
    while ((SPI0->SR & SPI_SR_RXNE) == 0) {
    }

    /* 读回从机数据 */
    return (uint8_t)SPI0->DR;
}
```

连续读写：

```c
void spi_write_read(
    const uint8_t *tx,
    uint8_t *rx,
    uint32_t len)
{
    for (uint32_t i = 0; i < len; i++) {
        rx[i] = spi_xfer(tx[i]);
    }
}
```

轮询的缺点与 UART 相同：`while(...)` 自旋消耗大量 CPU。

---

# 3. 硬件 SPI：中断方式

中断方式让 CPU 不必持续自旋。常见实现用两套方向的状态机：发（TX）与收（RX）。

伪寄存器补充位：

```c
#define SPI_CR_TXIE  (1U << 3)  /* 使能 TX 中断 */
#define SPI_CR_RXIE  (1U << 4)  /* 使能 RX 中断 */

#define SPI_TX_BUF_SIZE 128
#define SPI_RX_BUF_SIZE 128

static volatile uint8_t tx_buf[SPI_TX_BUF_SIZE];
static volatile uint8_t rx_buf[SPI_RX_BUF_SIZE];
static volatile uint32_t tx_head, tx_tail;
static volatile uint32_t rx_head, rx_tail;
```

## 应用层提交发送

```c
int spi_write_byte(uint8_t data)
{
    uint32_t next = (tx_head + 1) % SPI_TX_BUF_SIZE;
    if (next == tx_tail) {
        return 0;   /* 缓冲区满 */
    }
    tx_buf[tx_head] = data;
    tx_head = next;

    /* 开启 TX 中断 */
    SPI0->CR |= SPI_CR_TXIE;
    return 1;
}
```

## SPI ISR（收发共用一个中断向量的典型外设）

```c
void SPI0_IRQHandler(void)
{
    uint32_t sr = SPI0->SR;

    /* RX 非空：读回从机返回的数据 */
    if (sr & SPI_SR_RXNE) {
        uint8_t d = (uint8_t)SPI0->DR;

        uint32_t next = (rx_head + 1) % SPI_RX_BUF_SIZE;
        if (next != rx_tail) {
            rx_buf[rx_head] = d;
            rx_head = next;
        }
    }

    /* TX 空且有待发数据 */
    if (sr & SPI_SR_TXE) {
        if (tx_head != tx_tail) {
            SPI0->DR = tx_buf[tx_tail];
            tx_tail = (tx_tail + 1) % SPI_TX_BUF_SIZE;
        } else {
            /* 无数据时关闭 TX 中断，避免 IRQ 风暴 */
            SPI0->CR &= ~SPI_CR_TXIE;
        }
    }
}
```

> 全双工注意点：从机只有在被 CS 选中且主机持续给出 SCK 时才会回数据。若一次传输 RX 全为 `0xFF`，通常是因为 CS 未选中或主机没有提供足够的时钟周期。

---

# 4. 硬件 SPI + DMA

DMA 的作用仍是搬运，而不是让时钟更快：

```text
Memory
  │         Memory
  │           │
TX DMA        RX DMA
  │           │
  ▼           ▼
SPI DR  ◄───── SPI MISO
SPI MOSI ────► (从机)
```

## SPI TX DMA

```c
typedef struct {
    volatile uint32_t SRC;
    volatile uint32_t DST;
    volatile uint32_t LEN;
    volatile uint32_t CTRL;
    volatile uint32_t STATUS;
} dma_regs_t;

#define DMA0 ((dma_regs_t *)0x50000000UL)

#define DMA_CTRL_ENABLE     (1U << 0)
#define DMA_CTRL_SRC_INC    (1U << 1)
#define DMA_CTRL_DST_FIXED  (1U << 2)
#define DMA_CTRL_TXIE       (1U << 3)   /* TX 完成中断 */

#define DMA_STATUS_DONE     (1U << 0)
```

启动发送：

```c
void spi_dma_write(const uint8_t *buf, uint32_t len)
{
    /* 源=内存（递增），目的=SPI DR（固定） */
    DMA0->SRC = (uint32_t)buf;
    DMA0->DST = (uint32_t)&SPI0->DR;
    DMA0->LEN = len;

    DMA0->CTRL =
        DMA_CTRL_SRC_INC |
        DMA_CTRL_DST_FIXED |
        DMA_CTRL_TXIE |
        DMA_CTRL_ENABLE;

    /* SPI 打开 TX DMA 请求 */
    SPI0->CR |= (1U << 6);
}
```

## SPI RX DMA

```c
static uint8_t rx_dma_buf[128];

void spi_dma_rx_start(void)
{
    DMA0->SRC = (uint32_t)&SPI0->DR;      /* 固定   */
    DMA0->DST = (uint32_t)rx_dma_buf;     /* 递增   */
    DMA0->LEN = sizeof(rx_dma_buf);

    DMA0->CTRL =
        DMA_CTRL_DST_INC |
        DMA_CTRL_SRC_FIXED |
        DMA_CTRL_TXIE |
        DMA_CTRL_ENABLE;

    SPI0->CR |= (1U << 7);   /* SPI RX DMA 请求 */
}
```

## DMA 完成中断

```c
static volatile int spi_dma_busy;

void DMA0_IRQHandler(void)
{
    if (DMA0->STATUS & DMA_STATUS_DONE) {
        DMA0->STATUS = DMA_STATUS_DONE;   /* 清标志 */
        DMA0->CTRL &= ~DMA_CTRL_ENABLE;
        spi_dma_busy = 0;
    }
}
```

### 常见坑
1. **DMA 完成 ≠ 传输完成**：DMA 把数据搬进 SPI DR 后，SPI 移位寄存器可能仍在串行输出。真正的"发完"需等待 SPI 的 Busy 标志清零，或关闭 SPI 前查询移位寄存器空闲。
2. **RX 必须与 TX 同时跑**：SPI 全双工下，不提供时钟就没有数据。若只配 RX DMA 而没有 TX DMA，需要额外"喂"驱动时钟（通常写 `0xFF` 填充，或依赖上一个 TX 的 SCK）。
3. **CS 的软件时序**：很多 MCU 的 CS 并不由 SPI+DMA 自动管理，需要在启动 DMA 前手动拉低 CS、传输结束后拉高。

---

# 5. 四种方式对比

| 方式 | CPU 占用 | 实现复杂度 | 数据量 | 实时性 | 典型用途 |
| :--- | :--: | :--: | :--: | :--: | :--- |
| 软件模拟 SPI | 很高 | 中 | 很低 | 较差 | 无 SPI 外设、特殊 GPIO 复用、DEBUG |
| 硬件轮询 | 高 | 最低 | 低 | 简单直接 | 少量寄存器读写、初始化 |
| 硬件中断 | 中低 | 中 | 中 | 高 | 普通外设驱动、传感器 |
| SPI + DMA | 最低 | 高 | 大 | 高 | LCD 刷屏、SD 卡、大批量数据搬运 |

---

# 6. CPU 参与程度

软件模拟 SPI：

```text
每一拍
CPU 都要翻转 SCK 和 MOSI
```
一个字节（8 bit、MSB 先行）至少 8 次采样操作。

硬件轮询：

```text
每字节
CPU 忙等 TXE / RXNE
```
CPU 只在等待，没有释放。

硬件中断：

```text
每字节 / 每 FIFO 事件
进入 ISR
```

DMA：

```text
整个 Buffer
CPU 只需配置一次 + 等 DMA 完成
```

---

# 7. 抽象层级

软件模拟 SPI：

```text
Application
    ↓
GPIO 位移 + 翻转 SCK
    ↓
Pin
```

硬件 SPI：

```text
Application
    ↓
SPI Driver
    ↓
SPI Peripheral
    ↓
Pin
```

硬件 SPI + DMA：

```text
Application
    ↓
Buffer
    ↕
DMA
    ↕
SPI Peripheral
```

---

# 8. 总结

SPI 的实现本质与 UART 一样，区分"谁来产生时钟/时序"与"谁来搬运数据"：

```text
Bit-Bang：  时序=CPU 搬数据=CPU
Polling：   时序=SPI  搬数据=CPU 等待
IRQ：       时序=SPI  搬数据=CPU 被通知
DMA：       时序=SPI  搬数据=DMA / CPU 只做配置与收尾
```

> 与 UART 最大的不同：SPI 全双工、无帧起始/停止额外开销、速率可达 MHz 级；对应地，实现时必须先确认主从双方在 **CPOL/CPHA、位序（MSB 先行为主）、字长、CS 极性**上完全对齐，否则可能通信逻辑正确却数据错位。