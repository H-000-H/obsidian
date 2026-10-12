---
title: UART 驱动实现完整手册（Bit-Bang / 轮询 / 中断 / DMA）
tags:
  - Embedded
  - Protocol
  - UART
  - Driver
  - DMA
  - BitBanging
  - Python
  - C
status: Completed
created: 2026-09-20
---

# 📡 UART 驱动实现完整手册（Bit-Bang / 轮询 / 中断 / DMA）

> [!INFO] 笔记概览
> - **核心线索**：四种实现的本质区别只有两点——**谁产生 bit 时序**、**谁搬运 byte**。据此分为软件模拟、硬件轮询、硬件中断、UART+DMA。
> - **内容范围**：GPIO 位带模拟收发（Python）· 硬件阻塞轮询 · RX/TX 中断 + 环形缓冲 · DMA 搬运 · TC 语义（DMA 完成 ≠ 发送完成）· IDLE + DMA · Circular 模式 · 四种方式对比与 API 设计
> - **推荐路径**：§0 总览 → §1 Bit-Bang → §2 轮询 → §3 RX 中断 → §4 TX 中断 → §5~6 DMA → §7 IDLE → §9 对比 → §12 总结
> - **代码约定**：Python 用于 Bit-Bang 逻辑模拟（PEP8）；C 用于硬件外设驱动（**Allman 大括号**、4 空格缩进、控制语句强制花括号），均为教学用伪寄存器。

---

## 🗺️ 0. 四种实现方式总览

```text
软件模拟 UART ──► GPIO 手工产生 UART 时序，逐 bit 收发
硬件轮询 UART ──► CPU 查询状态寄存器（TX_EMPTY / RX_READY）阻塞等待
硬件中断 UART ──► UART 事件触发 IRQ ──► ISR 处理收发（配合环形缓冲）
UART + DMA   ──► UART 产生 DMA 请求 ──► DMA 自动搬运 ──► 完成后中断通知 CPU
```

> [!NOTE] 一句话记忆
> **Bit-Bang**：CPU 控 bit；**Polling**：CPU 等 byte；**IRQ**：UART 叫 CPU 搬 byte；**DMA**：DMA 自己搬 byte，CPU 只处理结果。

---

## 🖐️ 1. 软件模拟 UART（GPIO Bit-Banging）

不使用 MCU 内部 UART 外设，由 CPU 用 GPIO 手工画出时序。以最常用的 **8N1**（1 起始位 + 8 数据位 + 无校验 + 1 停止位、LSB 先行）为例。

```text
Idle(高)
  │   Start   Data[0..7]        Stop(高)
  │     │        │                 │
──┐     ┌──────────────────────────
  └─────┘ 0 1 0 1 1 0 1 0 ─────────
```

单个 bit 时间由波特率决定：

$$T_{\text{bit}} = \frac{1}{\text{baud}} \quad\Rightarrow\quad 9600\ \text{baud} \approx 104.167\ \mu s$$

### 1.1 软件发送（Python）

```python
import time

BAUD_RATE = 9600
BIT_TIME = 1.0 / BAUD_RATE


def gpio_write(level: int):
    """实际使用时替换为真实 GPIO 操作，例如 GPIO.output(TX_PIN, level)。"""
    print(level, end="")


def delay_bit():
    time.sleep(BIT_TIME)


def uart_send_byte(data: int):
    # UART 空闲状态为高电平
    gpio_write(0)          # Start Bit：拉低一个 bit 时间
    delay_bit()

    # Data Bit：UART 低位先发（LSB First）
    for i in range(8):
        bit = (data >> i) & 0x01
        gpio_write(bit)
        delay_bit()

    gpio_write(1)          # Stop Bit：拉高一个 bit 时间
    delay_bit()


def uart_send(data: bytes):
    gpio_write(1)          # 确保空闲高电平
    for byte in data:
        uart_send_byte(byte)


uart_send(b"ABC")
```

发送 `0x41`（`0100 0001b`）的实际位序：因 LSB First，先发 `1 0 0 0 0 0 1 0`，前后各夹 Start(0) / Stop(1)。

### 1.2 软件接收（Python）

接收比发送复杂：需在 Start 下降沿后**对齐到 bit 中央**再采样，避免边沿抖动误读。

```python
import time

BAUD_RATE = 9600
BIT_TIME = 1.0 / BAUD_RATE


def gpio_read() -> int:
    return 1


def uart_recv_byte():
    # 1. 等待 Start Bit（RX 从高变低）
    while gpio_read() == 1:
        pass

    # 2. 等半个 bit，进入 Start Bit 中央再次确认，滤除毛刺
    time.sleep(BIT_TIME / 2)
    if gpio_read() != 0:
        return None

    # 3. 再走一个 bit，对齐到 Data Bit 0 中央
    time.sleep(BIT_TIME)

    data = 0
    for i in range(8):
        bit = gpio_read()
        data |= bit << i       # LSB First，低位在低索引
        time.sleep(BIT_TIME)

    # 4. 检查 Stop Bit
    if gpio_read() != 1:
        print("Framing Error")
        return None
    return data
```

> [!CAUTION] 纯轮询接收的代价
> `while gpio_read(): pass` 会让 CPU 满载自旋。实际嵌入式里软件 UART 接收通常配合 **GPIO 边沿中断 + 定时器**：RX 下降沿触发 IRQ → 启动 Timer → 1.5 bit 后采 Data0 → 之后每隔 1 bit 采一次。帧格式与过采样原理见 [[UART]] §1、§2。

---

## 🔁 2. 硬件 UART：轮询方式

MCU 内部已有 UART 外设，CPU 通过不断读状态寄存器判断"能不能发 / 有没有收到"。

```text
CPU ──寄存器──► UART Peripheral ──┬── TX
                                  └── RX
```

伪寄存器定义（C，Allman 风格）：

```c
#include <stdint.h>

typedef struct
{
    volatile uint32_t STATUS;   /* 状态寄存器 */
    volatile uint32_t DATA;     /* 数据寄存器（Tx/Rx 共用） */
    volatile uint32_t BAUD;     /* 波特率寄存器 */
    volatile uint32_t CTRL;     /* 控制寄存器 */
} uart_regs_t;

#define UART0 ((uart_regs_t *)0x40000000UL)

/* STATUS 标志位 */
#define UART_STATUS_TX_EMPTY     (1U << 0)  /* 发送保持寄存器空，可写下一字节 */
#define UART_STATUS_RX_READY     (1U << 1)  /* 接收缓冲非空，可读 */
#define UART_STATUS_TX_COMPLETE  (1U << 2)  /* 发送完全结束（含移位寄存器 + Stop 位） */

/* CTRL 控制位 */
#define UART_CTRL_ENABLE         (1U << 0)  /* UART 使能 */
#define UART_CTRL_RX_IRQ         (1U << 1)  /* RX 中断使能 */
#define UART_CTRL_TX_IRQ         (1U << 2)  /* TX Empty 中断使能 */
#define UART_CTRL_TX_DMA         (1U << 4)  /* TX DMA 请求使能 */
#define UART_CTRL_RX_DMA         (1U << 5)  /* RX DMA 请求使能 */
```

### 2.1 轮询发送

```c
void uart_putc(uint8_t data)
{
    while ((UART0->STATUS & UART_STATUS_TX_EMPTY) == 0)
    {
        /* 等待 TX 保持寄存器空 */
    }
    UART0->DATA = data;
}

void uart_write(const uint8_t *buf, uint32_t len)
{
    for (uint32_t i = 0; i < len; i++)
    {
        uart_putc(buf[i]);
    }
}
```

调用：

```c
static const uint8_t msg[] = "Hello UART\r\n";
uart_write(msg, sizeof(msg) - 1);   /* 不含结尾 '\0' */
```

### 2.2 轮询接收（Echo）

```c
uint8_t uart_getc(void)
{
    while ((UART0->STATUS & UART_STATUS_RX_READY) == 0)
    {
        /* 等待 RX 数据 */
    }
    return (uint8_t)UART0->DATA;
}

/* 简单回显：读到什么就发回什么 */
for (;;)
{
    uint8_t ch = uart_getc();
    uart_putc(ch);
}
```

> [!WARNING] 轮询的缺点
> `while(...)` 自旋消耗大量 CPU，等待期间无法做其它事，且无缓冲——高波特率下极易丢字节。适合 Bootloader / Early Debug 等简单场景。

---

## 🔔 3. 硬件 UART：中断方式（RX）

中断让 CPU 不必持续自旋：UART 收到数据 → 置 RX flag → 触发 IRQ → ISR 把字节压入环形缓冲 → 应用层异步取用。

```text
CPU 正常执行其他程序
        ↓
UART 收到数据 → 置 RX_READY → IRQ
        ↓
CPU 进入 ISR → 读 DATA → 存 Ring Buffer → 退出
        ↓
应用层在主循环里按需读取
```

### 3.1 环形缓冲与初始化

```c
#include <stdint.h>

#define UART_RX_BUF_SIZE 128

static volatile uint8_t  rx_buf[UART_RX_BUF_SIZE];
static volatile uint32_t rx_head;   /* ISR 写入位置 */
static volatile uint32_t rx_tail;   /* 应用读取位置 */

void uart_init(void)
{
    UART0->BAUD = 115200;              /* 配置波特率 */
    UART0->CTRL |= UART_CTRL_ENABLE;   /* 使能 UART */
    UART0->CTRL |= UART_CTRL_RX_IRQ;   /* 使能 RX 中断 */
    /* 真实 MCU 还需 NVIC_EnableIRQ(UART0_IRQn); */
}
```

### 3.2 RX ISR

```c
void UART0_IRQHandler(void)
{
    if (UART0->STATUS & UART_STATUS_RX_READY)
    {
        uint8_t data = (uint8_t)UART0->DATA;

        uint32_t next = (rx_head + 1) % UART_RX_BUF_SIZE;
        if (next != rx_tail)             /* 缓冲未满才写入，满则丢弃本字节 */
        {
            rx_buf[rx_head] = data;
            rx_head = next;
        }
    }
}
```

### 3.3 应用层读取

```c
int uart_read_byte(uint8_t *data)
{
    if (rx_head == rx_tail)
    {
        return 0;                        /* 缓冲空，无数据 */
    }
    *data = rx_buf[rx_tail];
    rx_tail = (rx_tail + 1) % UART_RX_BUF_SIZE;
    return 1;
}

int main(void)
{
    uart_init();
    for (;;)
    {
        uint8_t ch;
        if (uart_read_byte(&ch))
        {
            uart_putc(ch);               /* 回显；CPU 空闲时可处理其它任务 */
        }
    }
}
```

> [!NOTE] 单生产者-单消费者无锁环形缓冲
> `rx_head` 只由 ISR 改、`rx_tail` 只由主循环改，配合 `volatile` 即可无锁安全。这是最经典的 SPSC 队列。环形缓冲与溢出根因见 [[DMA、Buffer 与流量控制]] §3、§4。

---

## 📤 4. 硬件 UART：中断方式（TX）

发送同样可用中断：应用把字节压入 TX 环形缓冲，TX Empty 中断逐个取出写入 DATA。

```text
Application → TX Buffer → (TX Empty IRQ) → ISR 取下一字节 → 写 UART DATA
```

### 4.1 TX 缓冲与提交

```c
#define UART_TX_BUF_SIZE 128

static volatile uint8_t  tx_buf[UART_TX_BUF_SIZE];
static volatile uint32_t tx_head;
static volatile uint32_t tx_tail;

int uart_write_byte(uint8_t data)
{
    uint32_t next = (tx_head + 1) % UART_TX_BUF_SIZE;
    if (next == tx_tail)
    {
        return 0;                        /* 缓冲区满 */
    }
    tx_buf[tx_head] = data;
    tx_head = next;

    UART0->CTRL |= UART_CTRL_TX_IRQ;     /* 开启 TX Empty 中断，触发首次发送 */
    return 1;
}
```

### 4.2 收发共用 ISR

```c
void UART0_IRQHandler(void)
{
    /* RX：收到字节压入接收缓冲 */
    if (UART0->STATUS & UART_STATUS_RX_READY)
    {
        uint8_t data = (uint8_t)UART0->DATA;
        uint32_t next = (rx_head + 1) % UART_RX_BUF_SIZE;
        if (next != rx_tail)
        {
            rx_buf[rx_head] = data;
            rx_head = next;
        }
    }

    /* TX：缓冲有数据就继续发，发完关闭 TX 中断 */
    if (UART0->STATUS & UART_STATUS_TX_EMPTY)
    {
        if (tx_head != tx_tail)
        {
            UART0->DATA = tx_buf[tx_tail];
            tx_tail = (tx_tail + 1) % UART_TX_BUF_SIZE;
        }
        else
        {
            UART0->CTRL &= ~UART_CTRL_TX_IRQ;   /* 无数据必须关 TX 中断 */
        }
    }
}
```

> [!CAUTION] TX 中断不能常开
> RX 中断可长期打开；但 TX Empty 中断若无数据仍开着，会因"TX 一直为空"不停触发 IRQ，形成 **IRQ 风暴** 拖垮 CPU。正确做法：**有数据才开 TX 中断，发完立即关**。ISR 下半部处理与背压见 [[DMA、Buffer 与流量控制]] §7。

---

## 🚀 5. UART + DMA（TX）

DMA 的目的**不是让 UART 更快**——线路波特率不变（115200 还是 115200），DMA 优化的是 **CPU 搬运数据的开销**。

```text
不用 DMA：Memory ─(CPU load)→ CPU ─(CPU store)→ UART DATA → TX   （每字节都占 CPU）
用 DMA： Memory ─────────────→ DMA ───────────→ UART DATA → TX   （CPU 只配置 + 收完成中断）
```

DMA 伪寄存器（补齐源/目的地址模式宏，与 [[SPI-code]] 保持一致）：

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

#define DMA_CTRL_ENABLE      (1U << 0)   /* 通道使能 */
#define DMA_CTRL_SRC_INC     (1U << 1)   /* 源地址递增 */
#define DMA_CTRL_SRC_FIXED   (1U << 2)   /* 源地址固定 */
#define DMA_CTRL_DST_INC     (1U << 3)   /* 目的地址递增 */
#define DMA_CTRL_DST_FIXED   (1U << 4)   /* 目的地址固定 */
#define DMA_CTRL_IRQ_ENABLE  (1U << 5)   /* 传输完成中断使能 */

#define DMA_STATUS_DONE      (1U << 0)   /* 传输完成标志 */
```

### 5.1 启动 TX DMA（内存 → UART DATA）

```c
void uart_dma_write(const uint8_t *buf, uint32_t len)
{
    DMA0->SRC = (uint32_t)buf;           /* 源 = 内存（递增） */
    DMA0->DST = (uint32_t)&UART0->DATA;  /* 目的 = UART DATA（固定） */
    DMA0->LEN = len;

    DMA0->CTRL =
        DMA_CTRL_SRC_INC |
        DMA_CTRL_DST_FIXED |
        DMA_CTRL_IRQ_ENABLE;

    UART0->CTRL |= UART_CTRL_TX_DMA;     /* UART 打开 TX DMA 请求 */
    DMA0->CTRL  |= DMA_CTRL_ENABLE;      /* 启动 DMA */
}
```

### 5.2 DMA 完成中断

```c
static volatile int uart_dma_busy;

void uart_dma_write_async(const uint8_t *buf, uint32_t len)
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

void DMA0_IRQHandler(void)
{
    if (DMA0->STATUS & DMA_STATUS_DONE)
    {
        DMA0->STATUS = DMA_STATUS_DONE;      /* 清标志 */
        DMA0->CTRL  &= ~DMA_CTRL_ENABLE;     /* 关通道 */
        uart_dma_busy = 0;
    }
}
```

---

## ⚠️ 6. DMA 完成 ≠ UART 真正发送完成

这是 UART DMA 最容易踩的坑。

```text
DMA Complete 的含义：最后一个 byte 已从 Memory 搬进 UART DATA / FIFO
但此刻 UART 内部可能仍在：FIFO → Shift Register → TX Pin 串行输出
```

$$\text{DMA Complete} \neq \text{UART Transmission Complete (TC)}$$

真正"发完"必须等 **UART TC（Transmission Complete）** 标志——它表示移位寄存器已把最后一个 Stop 位送出线路：

```c
/* DMA 完成后，若需确保线路彻底发完（如 RS-485 切收、进低功耗前）： */
while ((UART0->STATUS & UART_STATUS_TX_COMPLETE) == 0)
{
}
```

> [!WARNING] RS-485 方向切换必踩
> 半双工 RS-485 若在 DMA Complete 就立刻拉低 DE 切收，最后一个字节还没出线路就会被截断。必须等 TC。DE 切换"交接棒"时序见 [[modbus]] §5。

---

## 📥 7. UART RX DMA + IDLE 中断

RX 方向 DMA 把 UART DATA 自动搬进内存缓冲：

```text
RX Pin → UART → DATA Register → DMA → Memory Buffer
```

### 7.1 RX DMA 的固有问题

```c
static uint8_t rx_dma_buf[128];

void uart_dma_rx_start(void)
{
    DMA0->SRC = (uint32_t)&UART0->DATA;   /* 源 = UART DATA（固定） */
    DMA0->DST = (uint32_t)rx_dma_buf;     /* 目的 = 内存（递增） */
    DMA0->LEN = sizeof(rx_dma_buf);

    DMA0->CTRL =
        DMA_CTRL_SRC_FIXED |
        DMA_CTRL_DST_INC |
        DMA_CTRL_IRQ_ENABLE |
        DMA_CTRL_ENABLE;

    UART0->CTRL |= UART_CTRL_RX_DMA;      /* 打开 RX DMA 请求 */
}
```

问题：若缓冲设 128 字节，但对端只发来 `AT\r\n`（4 字节），**DMA 永远收不满 → 不产生 Complete 中断**，CPU 无从得知数据已到。

### 7.2 IDLE + DMA：高性能接收标配

解决办法是配合 **UART IDLE Line 检测**：总线空闲一拍即触发 IDLE IRQ，此时查询 DMA 剩余量算出实际收到长度。

```c
#define RX_BUF_SIZE 256
static uint8_t rx_buf[RX_BUF_SIZE];

void uart_rx_start(void)
{
    dma_rx_start(&UART0->DATA, rx_buf, RX_BUF_SIZE);
    uart_enable_idle_irq();
}

void UART0_IRQHandler(void)
{
    if (uart_is_idle())
    {
        uart_clear_idle();                          /* 清 IDLE 标志 */

        uint32_t remain   = dma_get_remaining();    /* DMA 还剩多少没搬 */
        uint32_t received = RX_BUF_SIZE - remain;   /* 已收长度 */

        dma_rx_stop();
        process_uart_data(rx_buf, received);        /* 交给协议层解析 */
        dma_rx_start(&UART0->DATA, rx_buf, RX_BUF_SIZE);  /* 重启接收 */
    }
}
```

这样即使只收到 10 字节，也能在总线空闲时立刻处理，不必等 256 字节填满。

> [!TIP] 帧边界不用 DMA TC
> 变长协议（如 Modbus RTU）绝不能用"DMA 收满"当帧边界，要用 **IDLE / 帧间隔 T3.5** 判帧。IDLE + DMA 的完整 RX 状态机见 [[Modbus-code]]、[[DMA、Buffer 与流量控制]] §2。

---

## 🔄 8. DMA Circular Mode

持续数据流可用 **Circular DMA**：写到缓冲末尾自动回到 0，形成环形，无需每次重启。

```text
UART RX → DMA → ┌─────────────────────┐
                │   Circular Buffer   │  0 → 128 → 255 → 0（自动回绕）
                └─────────────────────┘
```

通常配合三个事件使用：

```text
DMA Half Transfer (HT)   ── 搬到一半，处理前半区
DMA Transfer Complete(TC)── 搬满回绕，处理后半区
UART IDLE                ── 帧结束，处理当前已收部分
```

> [!NOTE] Circular 的防覆盖
> Circular 模式下写指针可能追上读指针造成覆盖，需做高/低水位检测。Ping-Pong、水位滞回与多核共享见 [[DMA、Buffer 与流量控制]] §5、§6、[[SPI-DMA-多核-Cache]]。

---

## 📊 9. 四种方式对比

| 方式 | CPU 占用 | 实现复杂度 | 适用数据量 | 实时性 | 常见用途 |
| :--- | :---: | :---: | :---: | :---: | :--- |
| 软件模拟 UART | 很高 | 中 | 很低 | 较差 | 无 UART 外设、调试、引脚复用冲突 |
| 硬件轮询 | 高 | 最低 | 低 | 简单直接 | Bootloader、Early Debug |
| 硬件中断 | 中低 | 中 | 中 | 高 | 普通串口通信、命令交互 |
| UART + DMA | 最低 | 高 | 大 | 高 | 高吞吐、持续数据流、日志 |

---

## 🧮 10. CPU 参与度与抽象层级

### 10.1 CPU 参与度对比

以发送 1024 字节为例：

```text
软件 UART：每个 bit 都参与，一字节 ≈ 10 bit（Start+8Data+Stop）→ CPU 处理约 10240 次时序
硬件轮询： 每字节参与，CPU 操作约 1024 次（查 TX_EMPTY + 写 DATA）
硬件中断： 每字节/每 FIFO 事件进 ISR，IRQ 约 1024 次（有 FIFO 可显著下降）
UART+DMA：配置 DMA + DMA Complete IRQ ≈ 2 次主要 CPU 操作
```

### 10.2 抽象层级

```text
软件 UART：  Application → GPIO → Pin
硬件轮询：   Application → UART Driver → UART Peripheral → Pin
硬件中断：   Application → Buffer ← UART ISR ← UART Peripheral
UART+DMA：   Application ↔ Buffer ↔ DMA ↔ UART Peripheral
```

---

## 🏗️ 11. 实际驱动结构与 API 设计

完整的 UART 驱动往往同时支持多种后端：

```c
uart_poll_write();  uart_poll_read();
uart_irq_write();   uart_irq_read();
uart_dma_write();   uart_dma_read();
```

上层可暴露统一 API，底层按配置/长度/同步语义选择后端：

```c
if (len <= UART_FIFO_SIZE)
{
    uart_poll_write(dev, buf, len);
}
else
{
    uart_dma_write(dev, buf, len);
}
```

> [!CAUTION] 不要无条件自动切换后端
> 同步/异步语义、completion callback、buffer 生命周期会因此变得不明确。更稳妥的做法是**显式区分阻塞与非阻塞**：
> ```c
> uart_write(dev, buf, len);         /* Polling / Blocking：返回即发完 */
> uart_write_async(dev, buf, len, cb, arg);  /* IRQ / DMA：立即返回，完成回调 */
> ```

---

## 🎯 12. 总结：四种实现的本质区别

| 实现 | bit 时序 | byte 搬运 | CPU 角色 |
| :--- | :---: | :---: | :--- |
| 软件模拟 UART | CPU | CPU | 全程参与每一 bit |
| 硬件轮询 | UART | CPU | 自旋等待（Polling） |
| 硬件中断 | UART | CPU | 被 IRQ 唤醒搬 byte |
| UART + DMA | UART | DMA | 配置 + 处理完成事件 |

一句话：**Bit-Bang** CPU 控 bit；**Polling** CPU 等 byte；**IRQ** UART 叫 CPU 搬 byte；**DMA** DMA 自己搬 byte、CPU 最后处理结果。

---

## 🔗 思考与延伸索引

- [[UART]] — UART 物理层、帧格式、波特率、过采样与校验的原理
- [[DMA、Buffer 与流量控制]] — 环形缓冲、溢出根因、背压与 TC 语义的系统讨论
- [[SPI-code]] — 同为"时序 + 搬运"四象限的 SPI 驱动实现，DMA 宏定义一致
- [[I2C-code]] — I²C 的 Bit-Bang / 轮询 / 中断 / DMA 四种实现对照
- [[Modbus-code]] — RS-485 半双工下 UART + DMA + IDLE 的完整帧收发状态机
- [[SPI-DMA-多核-Cache]] — DMA 与 Cache 一致性、多核共享 Buffer 同步

---

## 📚 参考资料

1. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（USART / DMA 章节，TC 与 IDLE 标志）.
2. **STMicroelectronics** — *AN4031: Using the STM32F4 DMA*（Circular / Double Buffer 模式）.
3. **Jack Ganssle** — *Techniques for Debouncing & Bit-Banged UART*（软件串口时序对齐）.
4. 项目内部笔记：[[UART]]、[[DMA、Buffer 与流量控制]]、[[Modbus-code]]。
