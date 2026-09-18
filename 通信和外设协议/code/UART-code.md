# UART 四种实现方式

UART 常见的软件实现方式可以分为：

1. 软件模拟 UART（Bit-Banging）
    
2. 硬件 UART + 轮询
    
3. 硬件 UART + 中断
    
4. 硬件 UART + DMA
    

整体关系：

```text
软件模拟 UART
    ↓
GPIO 手动产生 UART 时序

硬件轮询 UART
    ↓
CPU 查询 UART 状态寄存器

硬件中断 UART
    ↓
UART 事件触发 IRQ
    ↓
ISR 处理收发

UART + DMA
    ↓
UART 产生 DMA 请求
    ↓
DMA 自动搬运数据
    ↓
完成后中断通知 CPU
```

---

# 1. 软件模拟 UART

软件 UART 不使用 MCU 内部 UART 外设，而是通过 GPIO 手动模拟 UART 时序。

常见格式：

```text
Idle
  │
  │   Start     Data[0:7]              Stop
  │     │          │                     │
──┐     ┌─────────────────────────────────
  └─────┘ 0 1 0 1 1 0 1 0 ─────────────
```

以最常见的：

```text
8N1
```

为例：

- 1 个起始位
    
- 8 个数据位
    
- 无校验
    
- 1 个停止位
    
- LSB First
    

例如：

```text
波特率 = 9600 baud
```

单个 bit 时间：

```text
Tbit = 1 / 9600
     ≈ 104.167 us
```

## Python 软件 UART 发送示例

这里使用一个抽象的 GPIO 接口：

```python
import time


BAUD_RATE = 9600
BIT_TIME = 1.0 / BAUD_RATE


def gpio_write(level: int):
    """
    实际使用时替换为真实 GPIO 操作。

    例如：
    GPIO.output(TX_PIN, level)
    """
    print(level, end="")


def delay_bit():
    time.sleep(BIT_TIME)


def uart_send_byte(data: int):
    # UART 空闲状态为高电平

    # Start Bit
    gpio_write(0)
    delay_bit()

    # Data Bit
    # UART 一般低位先发
    for i in range(8):
        bit = (data >> i) & 0x01
        gpio_write(bit)
        delay_bit()

    # Stop Bit
    gpio_write(1)
    delay_bit()


def uart_send(data: bytes):
    gpio_write(1)

    for byte in data:
        uart_send_byte(byte)


uart_send(b"ABC")
```

发送一个字节：

```text
0x41
```

二进制：

```text
0100 0001
```

UART 实际发送顺序：

```text
Start
↓
0

Data
↓
1 0 0 0 0 0 1 0

Stop
↓
1
```

因为：

```text
LSB First
```

所以最低位先发送。

---

## Python 软件 UART 接收示例

软件接收比发送复杂。

基本过程：

```text
等待 RX 从 1 → 0
        ↓
检测 Start Bit
        ↓
等待 0.5 bit
        ↓
进入 bit 中央
        ↓
每隔 1 bit 采样
        ↓
读取 8 bit
        ↓
检测 Stop Bit
```

示例：

```python
import time


BAUD_RATE = 9600
BIT_TIME = 1.0 / BAUD_RATE


def gpio_read() -> int:
c
    return 1


def uart_recv_byte():
    # 等待 Start Bit
    while gpio_read() == 1:
        pass

    # 等待到 Start Bit 中间
    time.sleep(BIT_TIME / 2)

    # 再次确认仍然是低电平
    if gpio_read() != 0:
        return None

    # 移动到 Data Bit 0 中央
    time.sleep(BIT_TIME)

    data = 0

    for i in range(8):
        bit = gpio_read()

        data |= bit << i

        time.sleep(BIT_TIME)

    # Stop Bit
    stop_bit = gpio_read()

    if stop_bit != 1:
        print("Framing Error")
        return None

    return data
```

实际嵌入式系统里，软件 UART 接收通常不会简单使用：

```python
while gpio_read():
    pass
```

而通常配合：

```text
GPIO 边沿中断
+
定时器
```

工作方式：

```text
RX 下降沿
    ↓
GPIO IRQ
    ↓
启动 Timer
    ↓
1.5 bit 后采样 Data0
    ↓
每隔 1 bit 采样一次
```

否则 CPU 占用非常高。

---

# 2. 硬件 UART：轮询方式

MCU 内部已经存在 UART 外设：

```text
CPU
 │
 │ 寄存器
 ▼
UART Peripheral
 │
 ├── TX
 │
 └── RX
```

CPU 通过不断读取 UART 状态寄存器判断：

```text
能不能发送
有没有收到数据
```

例如 UART 内部可能具有：

```text
STATUS
DATA
BAUD
CTRL
```

等寄存器。

---

## 轮询发送

伪寄存器定义：

```c
#include <stdint.h>

typedef struct {
    volatile uint32_t STATUS;
    volatile uint32_t DATA;
    volatile uint32_t BAUD;
    volatile uint32_t CTRL;
} uart_regs_t;

#define UART0 ((uart_regs_t *)0x40000000UL)

#define UART_STATUS_TX_EMPTY (1U << 0)
#define UART_STATUS_RX_READY (1U << 1)
```

发送一个字节：

```c
void uart_putc(uint8_t data)
{
    while ((UART0->STATUS & UART_STATUS_TX_EMPTY) == 0) {
        /* 等待 TX 寄存器空 */
    }

    UART0->DATA = data;
}
```

发送字符串：

```c
void uart_write(const uint8_t *buf, uint32_t len)
{
    for (uint32_t i = 0; i < len; i++) {
        uart_putc(buf[i]);
    }
}
```

调用：

```c
static const uint8_t msg[] = "Hello UART\r\n";

uart_write(msg, sizeof(msg) - 1);
```

---

## 轮询接收

```c
uint8_t uart_getc(void)
{
    while ((UART0->STATUS & UART_STATUS_RX_READY) == 0) {
        /* 等待 RX 数据 */
    }

    return (uint8_t)UART0->DATA;
}
```

例如：

```c
for (;;) {
    uint8_t ch = uart_getc();

    uart_putc(ch);
}
```

实现：

```text
RX
 ↓
UART DATA
 ↓
CPU 轮询发现 RX_READY
 ↓
读取 DATA
 ↓
再写入 DATA
 ↓
TX
```

也就是简单 Echo。

---

# 3. 硬件 UART：中断方式

轮询存在一个明显问题：

```c
while (!(UART->STATUS & RX_READY)) {
}
```

CPU 一直被占用。

中断方式则变成：

```text
CPU 正常执行其他程序
        ↓

UART 收到数据
        ↓
UART 设置 RX flag
        ↓
IRQ
        ↓
CPU 进入 ISR
        ↓
读取数据
        ↓
退出 ISR
```

因此 CPU 不需要持续轮询 UART。

---

## RX 中断基本结构

定义缓冲区：

```c
#include <stdint.h>

#define UART_RX_BUF_SIZE 128

static volatile uint8_t rx_buf[UART_RX_BUF_SIZE];

static volatile uint32_t rx_head;
static volatile uint32_t rx_tail;
```

初始化：

```c
void uart_init(void)
{
    /*
     * 配置波特率
     */
    UART0->BAUD = 115200;

    /*
     * 使能 UART
     */
    UART0->CTRL |= (1U << 0);

    /*
     * 使能 RX 中断
     */
    UART0->CTRL |= (1U << 1);

    /*
     * 真实 MCU 中还需要：
     *
     * NVIC_EnableIRQ(UART0_IRQn);
     */
}
```

---

## UART ISR

```c
void UART0_IRQHandler(void)
{
    if (UART0->STATUS & UART_STATUS_RX_READY) {
        uint8_t data = (uint8_t)UART0->DATA;

        uint32_t next =
            (rx_head + 1) % UART_RX_BUF_SIZE;

        if (next != rx_tail) {
            rx_buf[rx_head] = data;
            rx_head = next;
        }
    }
}
```

这里实现的是：

```text
UART RX
  ↓
ISR
  ↓
Ring Buffer
  ↓
Application
```

---

## 应用层读取

```c
int uart_read_byte(uint8_t *data)
{
    if (rx_head == rx_tail) {
        return 0;
    }

    *data = rx_buf[rx_tail];

    rx_tail =
        (rx_tail + 1) % UART_RX_BUF_SIZE;

    return 1;
}
```

主循环：

```c
int main(void)
{
    uart_init();

    for (;;) {
        uint8_t ch;

        if (uart_read_byte(&ch)) {
            uart_putc(ch);
        }

        /*
         * CPU 可以处理其他任务
         */
    }
}
```

---

# 4. UART TX 中断发送

发送也可以使用中断。

传统轮询：

```text
CPU
 ↓
等 TX Empty
 ↓
写 DATA
 ↓
等 TX Empty
 ↓
写 DATA
```

TX 中断：

```text
Application
 ↓
TX Buffer
 ↓
UART TX Empty IRQ
 ↓
ISR 取下一个字节
 ↓
写 UART DATA
```

例如：

```c
#define UART_TX_BUF_SIZE 128

static volatile uint8_t tx_buf[UART_TX_BUF_SIZE];

static volatile uint32_t tx_head;
static volatile uint32_t tx_tail;
```

发送：

```c
int uart_write_byte(uint8_t data)
{
    uint32_t next =
        (tx_head + 1) % UART_TX_BUF_SIZE;

    if (next == tx_tail) {
        return 0;
    }

    tx_buf[tx_head] = data;
    tx_head = next;

    /*
     * 开启 TX Empty 中断
     */
    UART0->CTRL |= (1U << 2);

    return 1;
}
```

ISR：

```c
void UART0_IRQHandler(void)
{
    /*
     * RX
     */
    if (UART0->STATUS & UART_STATUS_RX_READY) {
        uint8_t data = (uint8_t)UART0->DATA;

        uint32_t next =
            (rx_head + 1) % UART_RX_BUF_SIZE;

        if (next != rx_tail) {
            rx_buf[rx_head] = data;
            rx_head = next;
        }
    }

    /*
     * TX
     */
    if (UART0->STATUS & UART_STATUS_TX_EMPTY) {
        if (tx_head != tx_tail) {
            UART0->DATA = tx_buf[tx_tail];

            tx_tail =
                (tx_tail + 1) % UART_TX_BUF_SIZE;
        } else {
            /*
             * 没数据以后关闭 TX Empty IRQ
             */
            UART0->CTRL &= ~(1U << 2);
        }
    }
}
```

注意：

```text
RX IRQ
```

一般可以长期打开。

但是：

```text
TX Empty IRQ
```

不能永久打开。

否则：

```text
TX 一直为空
    ↓
一直产生 IRQ
    ↓
CPU IRQ Storm
```

所以：

```text
有数据
    ↓
打开 TX IRQ

数据发完
    ↓
关闭 TX IRQ
```

---

# 5. UART + DMA

DMA 的目的不是：

```text
让 UART 传输更快
```

UART 在线路上的波特率仍然不变。

例如：

```text
115200 baud
```

用了 DMA 仍然是：

```text
115200 baud
```

DMA 优化的是：

```text
CPU 搬运数据的开销
```

---

## 不使用 DMA

```text
Memory
   │
   │ CPU load
   ▼
CPU
   │
   │ CPU store
   ▼
UART DATA
   │
   ▼
TX
```

每个字节都需要 CPU 参与。

---

## 使用 DMA

```text
Memory
   │
   │
   ▼
 DMA
   │
   ▼
UART DATA
   │
   ▼
TX
```

CPU 只需要：

```text
配置 DMA
    ↓
启动 DMA
    ↓
继续执行其他程序
    ↓
DMA Complete IRQ
```

---

# 6. UART TX DMA

假设存在一个抽象 DMA：

```c
typedef struct {
    volatile uint32_t SRC;
    volatile uint32_t DST;
    volatile uint32_t LEN;
    volatile uint32_t CTRL;
    volatile uint32_t STATUS;
} dma_regs_t;

#define DMA0 ((dma_regs_t *)0x50000000UL)

#define DMA_CTRL_ENABLE       (1U << 0)
#define DMA_CTRL_SRC_INC      (1U << 1)
#define DMA_CTRL_DST_FIXED    (1U << 2)
#define DMA_CTRL_IRQ_ENABLE   (1U << 3)

#define DMA_STATUS_DONE       (1U << 0)
```

启动 UART DMA：

```c
void uart_dma_write(
    const uint8_t *buf,
    uint32_t len)
{
    /*
     * DMA Source:
     *
     * Memory
     */
    DMA0->SRC = (uint32_t)buf;

    /*
     * DMA Destination:
     *
     * UART DATA Register
     */
    DMA0->DST =
        (uint32_t)&UART0->DATA;

    DMA0->LEN = len;

    /*
     * Memory 地址递增
     *
     * UART DATA 地址固定
     */
    DMA0->CTRL =
        DMA_CTRL_SRC_INC |
        DMA_CTRL_DST_FIXED |
        DMA_CTRL_IRQ_ENABLE;

    /*
     * UART 打开 TX DMA Request
     */
    UART0->CTRL |= (1U << 4);

    /*
     * Start DMA
     */
    DMA0->CTRL |= DMA_CTRL_ENABLE;
}
```

例如：

```c
static const uint8_t msg[] =
    "Hello DMA UART\r\n";

uart_dma_write(
    msg,
    sizeof(msg) - 1);
```

工作过程：

```text
Memory[0]
    ↓
DMA
    ↓
UART DATA
    ↓
TX

Memory[1]
    ↓
DMA
    ↓
UART DATA

...

Memory[N]
```

---

# 7. DMA 完成中断

```c
static volatile int uart_dma_busy;

void uart_dma_write(
    const uint8_t *buf,
    uint32_t len)
{
    uart_dma_busy = 1;

    DMA0->SRC = (uint32_t)buf;
    DMA0->DST = (uint32_t)&UART0->DATA;
    DMA0->LEN = len;

    DMA0->CTRL =
        DMA_CTRL_SRC_INC |
        DMA_CTRL_DST_FIXED |
        DMA_CTRL_IRQ_ENABLE |
        DMA_CTRL_ENABLE;
}
```

DMA ISR：

```c
void DMA0_IRQHandler(void)
{
    if (DMA0->STATUS & DMA_STATUS_DONE) {
        /*
         * Clear interrupt flag
         */
        DMA0->STATUS = DMA_STATUS_DONE;

        /*
         * Disable DMA
         */
        DMA0->CTRL &= ~DMA_CTRL_ENABLE;

        uart_dma_busy = 0;
    }
}
```

---

# 8. DMA 完成不代表 UART 真正发送完成

这是 UART DMA 很容易踩的坑。

DMA Complete 的含义通常只是：

```text
最后一个 byte
Memory
  ↓
UART DATA / FIFO
```

已经完成。

但 UART 内部可能仍然正在发送：

```text
UART FIFO
   ↓
Shift Register
   ↓
TX Pin
```

所以：

```text
DMA Complete
```

不一定等于：

```text
UART Transmission Complete
```

真正发送完成通常需要等待：

```text
UART TC
Transmission Complete
```

例如：

```c
while ((UART0->STATUS & UART_STATUS_TX_COMPLETE) == 0) {
}
```

实际过程：

```text
DMA Complete
     ↓
UART FIFO 可能还有数据
     ↓
Shift Register 继续输出
     ↓
最后 Stop Bit 完成
     ↓
UART TC = 1
```

---

# 9. UART RX DMA

UART RX DMA：

```text
RX Pin
   ↓
UART
   ↓
DATA Register
   ↓
DMA
   ↓
Memory Buffer
```

例如：

```c
static uint8_t rx_dma_buf[128];
```

启动：

```c
void uart_dma_rx_start(void)
{
    DMA0->SRC =
        (uint32_t)&UART0->DATA;

    DMA0->DST =
        (uint32_t)rx_dma_buf;

    DMA0->LEN =
        sizeof(rx_dma_buf);

    DMA0->CTRL =
        DMA_CTRL_SRC_FIXED |
        DMA_CTRL_DST_INC |
        DMA_CTRL_IRQ_ENABLE |
        DMA_CTRL_ENABLE;

    /*
     * Enable UART RX DMA request
     */
    UART0->CTRL |= UART_CTRL_RX_DMA;
}
```

DMA：

```text
UART DATA
 ↓
buf[0]

UART DATA
 ↓
buf[1]

UART DATA
 ↓
buf[2]

...
```

---

# 10. UART RX DMA 的问题

例如 DMA 缓冲区：

```text
128 byte
```

但是实际收到的数据：

```text
AT\r\n
```

只有：

```text
4 byte
```

那么 DMA：

```text
128 byte 没收满
```

就不会产生：

```text
DMA Complete
```

所以 RX DMA 经常需要配合：

```text
UART IDLE Line
```

---

# 11. UART IDLE + DMA

这是 MCU 中非常常见的高性能 UART 接收方式。

工作过程：

```text
UART RX
   ↓
DMA
   ↓
Buffer
   ↓
UART 总线出现空闲
   ↓
IDLE IRQ
   ↓
停止 / 查询 DMA
   ↓
计算实际收到长度
```

例如：

```c
#define RX_BUF_SIZE 256

static uint8_t rx_buf[RX_BUF_SIZE];
```

启动 DMA：

```c
void uart_rx_start(void)
{
    dma_rx_start(
        &UART0->DATA,
        rx_buf,
        RX_BUF_SIZE);

    uart_enable_idle_irq();
}
```

IDLE 中断：

```c
void UART0_IRQHandler(void)
{
    if (uart_is_idle()) {
        uart_clear_idle();

        uint32_t remain =
            dma_get_remaining();

        uint32_t received =
            RX_BUF_SIZE - remain;

        dma_rx_stop();

        process_uart_data(
            rx_buf,
            received);

        dma_rx_start(
            &UART0->DATA,
            rx_buf,
            RX_BUF_SIZE);
    }
}
```

这样即使只收到：

```text
10 byte
```

也不必等待：

```text
256 byte
```

全部填满。

---

# 12. DMA Circular Mode

持续 UART 数据流还可以使用：

```text
Circular DMA
```

结构：

```text
UART RX
   ↓
DMA
   ↓
┌─────────────────────┐
│                     │
│   Circular Buffer   │
│                     │
└─────────────────────┘
```

例如：

```text
Buffer Size = 256

0
↓
────────────────────────
        ↓
       128
────────────────────────
                    ↓
                   255
                    ↓
                    0
```

DMA 到末尾后自动：

```text
255 → 0
```

通常配合：

```text
DMA Half Transfer
DMA Transfer Complete
UART IDLE
```

使用。

---

# 13. 四种方式对比

|方式|CPU 占用|实现复杂度|数据量|实时性|常见用途|
|---|--:|--:|--:|--:|---|
|软件模拟 UART|很高|中|很低|较差|没有 UART 外设、调试|
|硬件轮询|高|最低|低|简单直接|Bootloader、Early Debug|
|硬件中断|中低|中|中|高|普通串口通信|
|UART + DMA|最低|高|大|高|高吞吐、持续数据流|

---

# 14. CPU 参与程度

软件 UART：

```text
每一个 Bit
CPU 都参与
```

例如一个字节：

```text
Start
+
8 Data
+
Stop

= 10 bit
```

CPU 至少需要处理：

```text
10 次时序
```

---

硬件轮询：

```text
每一个 Byte
CPU 参与
```

UART 硬件自己处理：

```text
Start
Data
Stop
```

CPU 只负责：

```text
DATA Register
```

---

硬件中断：

```text
每一个 Byte / FIFO Event
CPU 进入 ISR
```

UART：

```text
自动生成 Bit Stream
```

CPU：

```text
只处理数据
```

---

DMA：

```text
整个 Buffer
CPU 可能只参与一次
```

例如发送：

```text
1024 byte
```

轮询：

```text
CPU 操作约 1024 次
```

中断：

```text
IRQ 可能约 1024 次
```

DMA：

```text
配置 DMA
+
DMA Complete IRQ

≈ 2 次主要 CPU 操作
```

如果 UART 有 FIFO：

```text
中断次数
```

也可以进一步下降。

---

# 15. 抽象层级

从软件角度看：

```text
软件 UART
────────────────────
Application
    ↓
GPIO
    ↓
Pin
```

硬件 UART：

```text
Application
    ↓
UART Driver
    ↓
UART Peripheral
    ↓
Pin
```

UART IRQ：

```text
Application
    ↓
Buffer
    ↑
UART ISR
    ↑
UART Peripheral
```

UART DMA：

```text
Application
    ↓
Buffer
    ↕
DMA
    ↕
UART Peripheral
```

---

# 16. 实际驱动常见结构

比较完整的 UART Driver 往往同时支持：

```c
uart_poll_write();

uart_poll_read();

uart_irq_write();

uart_irq_read();

uart_dma_write();

uart_dma_read();
```

上层 API 甚至可以完全一致：

```c
uart_write(
    dev,
    buf,
    len);
```

底层根据：

```text
配置
数据长度
同步 / 异步模式
```

选择：

```text
Polling
IRQ
DMA
```

例如：

```c
if (len <= UART_FIFO_SIZE) {
    uart_poll_write(...);
} else {
    uart_dma_write(...);
}
```

但驱动层最好不要无条件自动切换，否则：

```text
同步语义
异步语义
completion callback
buffer 生命周期
```

容易变得不明确。

更常见的是显式区分：

```c
uart_write();
```

和：

```c
uart_write_async();
```

其中：

```text
uart_write
    ↓
Polling / Blocking

uart_write_async
    ↓
IRQ / DMA
```

---

# 17. 总结

UART 四种实现的本质区别主要是：

```text
谁负责产生 UART bit 时序
```

以及：

```text
谁负责搬运 byte
```

软件模拟 UART：

```text
Bit 时序：CPU
Byte 搬运：CPU
```

硬件轮询：

```text
Bit 时序：UART
Byte 搬运：CPU
CPU 等待方式：Polling
```

硬件中断：

```text
Bit 时序：UART
Byte 搬运：CPU
CPU 通知方式：IRQ
```

UART + DMA：

```text
Bit 时序：UART
Byte 搬运：DMA
CPU：配置 + Completion
```

可以简单记成：

```text
Bit-Bang
CPU 控 Bit

Polling
CPU 等 Byte

IRQ
UART 叫 CPU 搬 Byte

DMA
DMA 自己搬 Byte
CPU 最后处理结果
```