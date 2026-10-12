---
title: I²C 驱动实现完整手册（Bit-Bang / 轮询 / 中断 / DMA）
tags:
  - Embedded
  - Protocol
  - I2C
  - Driver
  - DMA
  - BitBanging
  - Python
  - C
status: Completed
created: 2026-09-20
---

# 🔗 I²C 驱动实现完整手册（Bit-Bang / 轮询 / 中断 / DMA）

> [!INFO] 笔记概览
> - **核心线索**：与 SPI/UART 一致，I²C 驱动按"谁产生时序 / 谁搬运数据"分为软件模拟、硬件轮询、硬件中断、DMA 四种方式。
> - **内容范围**：GPIO 位带模拟（START/STOP/ACK）· 硬件阻塞轮询读写 · 中断状态机（EV/ER）· DMA + LAST 位自动 NACK · 单字节读边界处理
> - **推荐路径**：§1 Bit-Bang → §2 轮询 → §3 中断 → §4 DMA → §5 对比
> - **代码约定**：Python 用于 GPIO 位带逻辑模拟（PEP8）；C 用于硬件外设驱动（**Allman 大括号**、4 空格缩进、控制语句强制花括号），均为教学用伪寄存器。

---

## 🗺️ 0. 四种实现方式总览

```text
软件模拟 I²C ──► GPIO 手工产生 START/STOP/SCL，逐位收发 + 处理 ACK
硬件轮询 I²C ──► CPU 查询 SR1/SR2（SB/ADDR/TXE/RXNE/BTF）阻塞等待
硬件中断 I²C ──► EV/ER 事件中断驱动状态机，非阻塞
I²C + DMA  ──► DMA 接管数据搬运，中断只处理 SB/ADDR 与完成收尾
```

---

## 🖐️ 1. 软件轮询模拟 I²C（GPIO Bit-Banging）

不使用 I²C 外设，由 CPU 用 GPIO 手工产生时序。核心：**SCL 高时 SDA 跳变 = 控制信号；SCL 低时才能改数据**。

```python
import time

# 延时函数（单片机中常用微秒延时 __delay_us 或 nop）
def I2cDelay():
    time.sleep(0.000005)  # 约 5 微秒，对应 ~100kHz 速率

# 基础 GPIO 操作接口（根据实际平台实现）
def GpioSdaHigh(): pass
def GpioSdaLow():  pass
def GpioSclHigh(): pass
def GpioSclLow():  pass
def GpioSdaRead() -> int: return 1  # 读 SDA 输入引脚电平 (0 或 1)


# ----------------------------------------------------
# 1. 初始化：空闲状态 SDA/SCL 均由上拉电阻拉高
# ----------------------------------------------------
def I2cInit():
    GpioSdaHigh()
    GpioSclHigh()
    I2cDelay()


# ----------------------------------------------------
# 2. 起始信号 (START)：SCL 为高时，SDA 由高变低
# ----------------------------------------------------
def I2cStart():
    GpioSdaHigh()
    GpioSclHigh()
    I2cDelay()
    GpioSdaLow()    # SCL 高电平期间，SDA 下降沿
    I2cDelay()
    GpioSclLow()    # 钳住 SCL，准备后续发送数据
    I2cDelay()


# ----------------------------------------------------
# 3. 重复起始信号 (RESTART)：读写方向切换而不释放总线
# ----------------------------------------------------
def I2cRestart():
    GpioSdaHigh()
    GpioSclHigh()
    I2cDelay()
    GpioSdaLow()
    I2cDelay()
    GpioSclLow()
    I2cDelay()


# ----------------------------------------------------
# 4. 停止信号 (STOP)：SCL 为高时，SDA 由低变高
# ----------------------------------------------------
def I2cStop():
    GpioSclLow()
    GpioSdaLow()    # 先把 SDA 拉低，为后面的上升沿做准备
    I2cDelay()
    GpioSclHigh()   # SCL 拉高
    I2cDelay()
    GpioSdaHigh()   # SCL 高电平期间，SDA 上升沿
    I2cDelay()


# ----------------------------------------------------
# 5. 发送一个字节 (Write Byte) + 接收应答 (ACK)
# 返回值：True 表示收到从机 ACK(0)，False 表示 NACK(1)
# ----------------------------------------------------
def I2cWriteByte(byte_data: int) -> bool:
    # 1. 串行发送 8 位数据 (MSB 高位先行)
    for i in range(8):
        # 必须在 SCL 为低期间改变 SDA
        if (byte_data & 0x80) != 0:
            GpioSdaHigh()
        else:
            GpioSdaLow()

        byte_data <<= 1
        I2cDelay()

        GpioSclHigh()       # 产生 SCL 上升沿并保持高，让从机采样
        I2cDelay()
        GpioSclLow()        # 拉低 SCL，准备下一 bit
        I2cDelay()

    # 2. 接收从机应答（第 9 个时钟周期）：主机释放 SDA，让从机控制
    GpioSdaHigh()
    I2cDelay()
    GpioSclHigh()           # SCL 拉高供主机采样 ACK
    I2cDelay()
    ack = GpioSdaRead()     # 读取从机应答：0 为 ACK，1 为 NACK
    GpioSclLow()            # 结束第 9 个脉冲
    I2cDelay()

    return (ack == 0)


# ----------------------------------------------------
# 6. 读取一个字节 (Read Byte) + 发送应答
# send_ack: True 回复 ACK(继续读)，False 回复 NACK(停止读)
# ----------------------------------------------------
def I2cReadByte(send_ack: bool) -> int:
    byte_data = 0
    GpioSdaHigh()           # 主机释放 SDA，准备读取

    # 1. 串行读取 8 位
    for i in range(8):
        byte_data <<= 1
        GpioSclLow()
        I2cDelay()
        GpioSclHigh()       # SCL 拉高，此时从机已准备好数据
        I2cDelay()
        if GpioSdaRead():
            byte_data |= 0x01

    GpioSclLow()
    I2cDelay()

    # 2. 主机发送应答给从机（第 9 个时钟周期）
    if send_ack:
        GpioSdaLow()        # 发送 ACK (0)
    else:
        GpioSdaHigh()       # 发送 NACK (1)

    I2cDelay()
    GpioSclHigh()           # SCL 脉冲，让从机读取应答
    I2cDelay()
    GpioSclLow()
    I2cDelay()

    GpioSdaHigh()           # 释放 SDA
    return byte_data
```

> [!NOTE] Bit-Bang 关键点
> 开漏总线下"拉高"实为释放（靠上拉电阻拉高），因此读 ACK / 读数据前必须先 `GpioSdaHigh()` 释放 SDA。START/STOP/RESTART 时序与仲裁见 [[I2C]] §3、§5。

---

## 🔁 2. 硬件 I²C：阻塞轮询

CPU 通过查询状态寄存器完成收发，每步都带超时保护。

```c
#include <stdint.h>
#include <stdbool.h>

/* 硬件 I2C 寄存器结构体映射（以常见硬件外设为例） */
typedef struct
{
    volatile uint32_t CR1;      /* 控制寄存器 1 (使能、START、STOP、ACK控制) */
    volatile uint32_t CR2;      /* 控制寄存器 2 (时钟频率、中断、DMA 等) */
    volatile uint32_t OAR1;     /* 自身地址寄存器 */
    volatile uint32_t DR;       /* 数据寄存器 (Tx/Rx) */
    volatile uint32_t SR1;      /* 状态寄存器 1 (事件标志位) */
    volatile uint32_t SR2;      /* 状态寄存器 2 (系统状态标志位) */
} Hardware_I2C_TypeDef;

/* 状态寄存器 SR1 常用标志位掩码 */
#define I2C_SR1_SB      (1U << 0)   /* Start Bit: 起始位已发送 */
#define I2C_SR1_ADDR    (1U << 1)   /* Address Sent: 地址发送完成且收到 ACK */
#define I2C_SR1_BTF     (1U << 2)   /* Byte Transfer Finished: 字节传输完成 */
#define I2C_SR1_RXNE    (1U << 6)   /* Rx Data Register Not Empty: 读缓冲非空 */
#define I2C_SR1_TXE     (1U << 7)   /* Tx Data Register Empty: 发送缓冲空 */
#define I2C_SR1_AF      (1U << 10)  /* Acknowledge Failure: 从机应答失败(NACK) */

/* 状态寄存器 SR2 标志位 */
#define I2C_SR2_BUSY    (1U << 1)   /* Bus Busy: 总线正忙 */

/* 控制寄存器 CR1 控制位 */
#define I2C_CR1_START   (1U << 8)   /* 生成 START 信号 */
#define I2C_CR1_STOP    (1U << 9)   /* 生成 STOP 信号 */
#define I2C_CR1_ACK     (1U << 10)  /* 使能 ACK 应答 */

/**
 * @brief 硬件 I2C 阻塞轮询写入
 * @param i2c       外设基地址
 * @param dev_addr  7 位从机设备地址
 * @param data      待发送数据缓冲区
 * @param len       数据长度
 * @return true 写入成功, false 超时或无应答 (NACK)
 */
bool I2C_Master_Transmit_Poll(Hardware_I2C_TypeDef *i2c, uint8_t dev_addr,
                              const uint8_t *data, uint16_t len)
{
    uint32_t timeout;

    /* 1. 等待总线空闲 (Bus not busy) */
    timeout = 100000;
    while ((i2c->SR2 & I2C_SR2_BUSY) && (--timeout))
    {
    }
    if (timeout == 0)
    {
        return false;
    }

    /* 2. 硬件产生 START 信号 */
    i2c->CR1 |= I2C_CR1_START;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_SB) && (--timeout))   /* 轮询 SB 标志置位 */
    {
    }
    if (timeout == 0)
    {
        return false;
    }

    /* 3. 发送 7 位地址 + 写方向位 (0) */
    i2c->DR = (uint8_t)(dev_addr << 1) | 0x00;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_ADDR) && (--timeout))  /* 轮询从机是否回 ACK */
    {
        if (i2c->SR1 & I2C_SR1_AF)                     /* 收到 NACK */
        {
            i2c->CR1 |= I2C_CR1_STOP;
            i2c->SR1 &= ~I2C_SR1_AF;                   /* 清除失败标志 */
            return false;
        }
    }
    if (timeout == 0)
    {
        return false;
    }

    /* 清除 ADDR 标志位（硬件标准流程：先读 SR1，再读 SR2） */
    (void)i2c->SR1;
    (void)i2c->SR2;

    /* 4. 轮询发送数据载荷 */
    for (uint16_t i = 0; i < len; ++i)
    {
        timeout = 100000;
        while (!(i2c->SR1 & I2C_SR1_TXE) && (--timeout))  /* 等待发送缓冲空 */
        {
        }
        if (timeout == 0)
        {
            return false;
        }
        i2c->DR = data[i];                                /* 写入硬件移位器 */
    }

    /* 5. 等待最后一个字节完全移出总线 (BTF) */
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_BTF) && (--timeout))
    {
    }
    if (timeout == 0)
    {
        return false;
    }

    /* 6. 产生 STOP 信号释放总线 */
    i2c->CR1 |= I2C_CR1_STOP;
    return true;
}

/**
 * @brief 硬件 I2C 阻塞轮询读取
 * @param i2c       外设基地址
 * @param dev_addr  7 位从机设备地址
 * @param buf       接收缓冲区
 * @param len       需要读取的字节数
 * @return true 读取成功, false 失败
 */
bool I2C_Master_Receive_Poll(Hardware_I2C_TypeDef *i2c, uint8_t dev_addr,
                             uint8_t *buf, uint16_t len)
{
    uint32_t timeout;
    if (len == 0)
    {
        return false;
    }

    /* 1. 等待总线空闲 */
    timeout = 100000;
    while ((i2c->SR2 & I2C_SR2_BUSY) && (--timeout))
    {
    }
    if (timeout == 0)
    {
        return false;
    }

    i2c->CR1 |= I2C_CR1_ACK;        /* 使能硬件自动回复 ACK */

    /* 2. 产生 START */
    i2c->CR1 |= I2C_CR1_START;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_SB) && (--timeout))
    {
    }
    if (timeout == 0)
    {
        return false;
    }

    /* 3. 发送 7 位地址 + 读方向位 (1) */
    i2c->DR = (uint8_t)(dev_addr << 1) | 0x01;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_ADDR) && (--timeout))
    {
    }
    if (timeout == 0)
    {
        return false;
    }

    /* 特殊边界：只读 1 个字节 */
    if (len == 1)
    {
        /* 硬件规定：单字节读取需在清 ADDR 前先关 ACK，并请求 STOP */
        i2c->CR1 &= ~I2C_CR1_ACK;
        (void)i2c->SR1;
        (void)i2c->SR2;
        i2c->CR1 |= I2C_CR1_STOP;

        timeout = 100000;
        while (!(i2c->SR1 & I2C_SR1_RXNE) && (--timeout))
        {
        }
        if (timeout == 0)
        {
            return false;
        }
        buf[0] = (uint8_t)i2c->DR;
        return true;
    }

    /* 多字节：清除 ADDR 标志进入接收 */
    (void)i2c->SR1;
    (void)i2c->SR2;

    /* 4. 循环读取 */
    for (uint16_t i = 0; i < len; ++i)
    {
        /* 倒数第 1 个字节时，提前置 NACK 并发 STOP */
        if (i == len - 1)
        {
            i2c->CR1 &= ~I2C_CR1_ACK;
            i2c->CR1 |= I2C_CR1_STOP;
        }

        timeout = 100000;
        while (!(i2c->SR1 & I2C_SR1_RXNE) && (--timeout))
        {
        }
        if (timeout == 0)
        {
            return false;
        }
        buf[i] = (uint8_t)i2c->DR;
    }

    return true;
}
```

> [!CAUTION] 单字节读取边界
> 读 1 字节时必须在**清除 ADDR 之前**关闭 ACK 并请求 STOP，否则从机会因收到 ACK 继续驱动 SDA，导致总线锁死。多字节读则在倒数第 1 字节前提前置 NACK。

---

## 🔔 3. 硬件 I²C：中断状态机

用句柄管理非阻塞传输，EV 中断按事件推进状态机，ER 中断处理错误。

```c
#include <stdint.h>
#include <stdbool.h>

typedef enum
{
    I2C_STATE_IDLE = 0,
    I2C_STATE_BUSY_TX,
    I2C_STATE_BUSY_RX,
    I2C_STATE_COMPLETE,
    I2C_STATE_ERROR
} I2C_State_t;

typedef struct
{
    Hardware_I2C_TypeDef *instance;   /* I2C 外设基地址 */
    uint8_t               dev_addr;   /* 7 位设备地址 */
    uint8_t              *buffer;     /* 数据缓冲区指针 */
    volatile uint16_t     length;     /* 总字节数 */
    volatile uint16_t     index;      /* 当前已传输索引 */
    volatile I2C_State_t  state;      /* 传输状态机 */
    volatile uint32_t     error_code;
} I2C_Handle_t;

static I2C_Handle_t g_i2c1_handle;

/* 中断使能控制位 (CR2 寄存器) */
#define I2C_CR2_ITERREN   (1U << 8)   /* 错误中断使能 */
#define I2C_CR2_ITEVTEN   (1U << 9)   /* 事件中断使能 */
#define I2C_CR2_ITBUFEN   (1U << 10)  /* 缓冲中断使能 (TXE/RXNE) */

/**
 * @brief 中断模式启动数据发送（非阻塞）
 */
bool I2C_Master_Transmit_IT(I2C_Handle_t *hi2c, uint8_t dev_addr,
                            const uint8_t *data, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0)
    {
        return false;
    }

    hi2c->dev_addr = dev_addr;
    hi2c->buffer   = (uint8_t *)data;
    hi2c->length   = len;
    hi2c->index    = 0;
    hi2c->state    = I2C_STATE_BUSY_TX;

    /* 开启事件、缓冲、错误中断 */
    hi2c->instance->CR2 |= (I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
    /* 硬件发送 START，触发第一次 EV 中断 (SB) */
    hi2c->instance->CR1 |= I2C_CR1_START;
    return true;
}

/**
 * @brief 中断模式启动数据接收（非阻塞）
 */
bool I2C_Master_Receive_IT(I2C_Handle_t *hi2c, uint8_t dev_addr,
                           uint8_t *buf, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0)
    {
        return false;
    }

    hi2c->dev_addr = dev_addr;
    hi2c->buffer   = buf;
    hi2c->length   = len;
    hi2c->index    = 0;
    hi2c->state    = I2C_STATE_BUSY_RX;

    hi2c->instance->CR1 |= I2C_CR1_ACK;   /* 开启应答 ACK */
    hi2c->instance->CR2 |= (I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
    hi2c->instance->CR1 |= I2C_CR1_START;
    return true;
}

/* 事件中断：按 SB → ADDR → TXE/BTF/RXNE 推进状态机 */
void I2C1_EV_IRQHandler(void)
{
    I2C_Handle_t *hi2c = &g_i2c1_handle;
    uint32_t sr1 = hi2c->instance->SR1;
    uint32_t sr2 = hi2c->instance->SR2;

    /* 1. SB: 起始条件产生 → 发地址 */
    if (sr1 & I2C_SR1_SB)
    {
        uint8_t rw_bit = (hi2c->state == I2C_STATE_BUSY_RX) ? 0x01 : 0x00;
        hi2c->instance->DR = (uint8_t)(hi2c->dev_addr << 1) | rw_bit;
        return;   /* 写 DR 自动清除 SB */
    }

    /* 2. ADDR: 地址发送完毕且收到从机 ACK */
    if (sr1 & I2C_SR1_ADDR)
    {
        if (hi2c->state == I2C_STATE_BUSY_RX)
        {
            /* 单字节接收特殊处理：清 ADDR 前关 ACK 并请求 STOP */
            if (hi2c->length == 1)
            {
                hi2c->instance->CR1 &= ~I2C_CR1_ACK;
                hi2c->instance->CR1 |= I2C_CR1_STOP;
            }
        }
        (void)sr1;   /* 读 SR1 再读 SR2 清除 ADDR 标志 */
        (void)sr2;
        return;
    }

    /* 3. TXE: 发送缓冲空（写数据流） */
    if ((sr1 & I2C_SR1_TXE) && (hi2c->state == I2C_STATE_BUSY_TX))
    {
        if (hi2c->index < hi2c->length)
        {
            hi2c->instance->DR = hi2c->buffer[hi2c->index++];
        }
        else
        {
            /* 数据已全部送入移位寄存器，关 ITBUFEN 防反复触发 TXE，
               剩下交给 BTF 确认最后字节完全飞出总线 */
            hi2c->instance->CR2 &= ~I2C_CR2_ITBUFEN;
        }
        return;
    }

    /* 4. BTF: 字节传输完全结束（发送尾部） */
    if ((sr1 & I2C_SR1_BTF) && (hi2c->state == I2C_STATE_BUSY_TX))
    {
        if (hi2c->index >= hi2c->length)
        {
            hi2c->instance->CR1 |= I2C_CR1_STOP;   /* 产生 STOP 释放总线 */
            hi2c->instance->CR2 &= ~(I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
            hi2c->state = I2C_STATE_COMPLETE;
            /* I2C_TxCpltCallback(hi2c);  回调用户或发信号量给应用任务 */
        }
        return;
    }

    /* 5. RXNE: 接收寄存器非空（读数据流） */
    if ((sr1 & I2C_SR1_RXNE) && (hi2c->state == I2C_STATE_BUSY_RX))
    {
        hi2c->buffer[hi2c->index++] = (uint8_t)hi2c->instance->DR;

        if (hi2c->index == (hi2c->length - 1))
        {
            /* 倒数第 1 字节到来前，提前关 ACK 并发 STOP */
            hi2c->instance->CR1 &= ~I2C_CR1_ACK;
            hi2c->instance->CR1 |= I2C_CR1_STOP;
        }
        else if (hi2c->index >= hi2c->length)
        {
            hi2c->instance->CR2 &= ~(I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
            hi2c->state = I2C_STATE_COMPLETE;
            /* I2C_RxCpltCallback(hi2c); */
        }
        return;
    }
}

/* 错误中断：处理 AF(NACK) 与 BERR(总线错误) */
void I2C1_ER_IRQHandler(void)
{
    I2C_Handle_t *hi2c = &g_i2c1_handle;
    uint32_t sr1 = hi2c->instance->SR1;

    if (sr1 & I2C_SR1_AF)                    /* 从机没有应答 (NACK) */
    {
        hi2c->instance->CR1 |= I2C_CR1_STOP; /* 释放总线 */
        hi2c->instance->SR1 &= ~I2C_SR1_AF;  /* 写 0 清除标志位 */
        hi2c->error_code |= 0x04;
    }

    if (sr1 & (1U << 8))                     /* BERR: 错误时刻出现 START/STOP 沿 */
    {
        hi2c->instance->SR1 &= ~(1U << 8);
        hi2c->error_code |= 0x01;
    }

    hi2c->instance->CR2 &= ~(I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
    hi2c->state = I2C_STATE_ERROR;
    /* I2C_ErrorCallback(hi2c); */
}
```

---

## 🚀 4. 硬件 I²C：DMA + 中断

DMA 接管数据搬运，中断只处理 SB/ADDR 与完成收尾。关键在接收用 **LAST 位**自动在最后一字节产生 NACK。

```c
#include <stdint.h>
#include <stdbool.h>

/* 控制寄存器 CR2 的 DMA 相关位 */
#define I2C_CR2_DMAEN     (1U << 11)  /* DMA 请求使能：TXE/RXNE 置位即向 DMA 发请求 */
#define I2C_CR2_LAST      (1U << 12)  /* 接收模式：指示下一次 DMA 传输是最后字节(产生 NACK) */

/* DMA 通道寄存器结构体（概念映射） */
typedef struct
{
    volatile uint32_t CCR;    /* 控制寄存器 (使能、方向、自增、中断使能) */
    volatile uint32_t CNDTR;  /* 传输计数器 (字节数，每发一个减 1) */
    volatile uint32_t CPAR;   /* 外设地址 (&i2c->DR) */
    volatile uint32_t CMAR;   /* 内存地址 (SRAM 缓冲区) */
} Hardware_DMA_Channel_TypeDef;

/**
 * @brief 启动 I2C DMA 主机发送
 */
bool I2C_Master_Transmit_DMA(I2C_Handle_t *hi2c, Hardware_DMA_Channel_TypeDef *dma_tx,
                             uint8_t dev_addr, const uint8_t *data, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0)
    {
        return false;
    }

    hi2c->state = I2C_STATE_BUSY_TX;
    hi2c->dev_addr = dev_addr;

    /* 1. 配置 DMA 通道 */
    dma_tx->CCR &= ~1U;                            /* 关闭通道准备配置 */
    dma_tx->CPAR = (uint32_t)&hi2c->instance->DR;  /* 外设目标：I2C 数据寄存器 */
    dma_tx->CMAR = (uint32_t)data;                 /* 内存源：数据首地址 */
    dma_tx->CNDTR = len;                           /* 传输长度 */
    dma_tx->CCR |= (1U << 4);                      /* 方向：Memory -> Peripheral */
    dma_tx->CCR |= (1U << 7);                      /* 内存地址自增 */
    dma_tx->CCR |= (1U << 1);                      /* 开启传输完成中断 (TCIE) */
    dma_tx->CCR |= 1U;                             /* 使能 DMA 通道 */

    /* 2. 使能 I2C 的 DMA 请求 */
    hi2c->instance->CR2 |= I2C_CR2_DMAEN;

    /* 3. 硬件产生 START（必须先发 START 和寻址） */
    hi2c->instance->CR1 |= I2C_CR1_START;

    /* 4. 事件中断只处理 SB 和 ADDR，之后 DMA 接管全部数据搬运 */
    hi2c->instance->CR2 |= I2C_CR2_ITEVTEN;
    return true;
}

/**
 * @brief 启动 I2C DMA 主机接收
 */
bool I2C_Master_Receive_DMA(I2C_Handle_t *hi2c, Hardware_DMA_Channel_TypeDef *dma_rx,
                            uint8_t dev_addr, uint8_t *buf, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0)
    {
        return false;
    }

    hi2c->state = I2C_STATE_BUSY_RX;
    hi2c->dev_addr = dev_addr;

    /* 1. 配置 DMA RX 通道 */
    dma_rx->CCR &= ~1U;
    dma_rx->CPAR = (uint32_t)&hi2c->instance->DR;
    dma_rx->CMAR = (uint32_t)buf;
    dma_rx->CNDTR = len;
    dma_rx->CCR &= ~(1U << 4);                     /* 方向：Peripheral -> Memory */
    dma_rx->CCR |= (1U << 7);                      /* 内存地址自增 */
    dma_rx->CCR |= (1U << 1);                      /* 开启完成中断 (TCIE) */
    dma_rx->CCR |= 1U;                             /* 使能 DMA 通道 */

    /* 2. 使能 ACK 并开启 DMA 请求 */
    hi2c->instance->CR1 |= I2C_CR1_ACK;
    hi2c->instance->CR2 |= I2C_CR2_DMAEN;

    /* 关键：使能 LAST 位——DMA 倒数到最后 1 次传输时，
       I2C 硬件自动将 ACK 置 0（发送 NACK） */
    if (len > 1)
    {
        hi2c->instance->CR2 |= I2C_CR2_LAST;
    }

    /* 3. 产生 START，进入中断处理 SB 和 ADDR */
    hi2c->instance->CR1 |= I2C_CR1_START;
    hi2c->instance->CR2 |= I2C_CR2_ITEVTEN;
    return true;
}

/* DMA 发送完成中断 */
void DMA1_Channel_I2C_TX_IRQHandler(void)
{
    DMA1->IFCR |= (1U << 1);                       /* 清除 DMA 传输完成标志 (TCIF) */
    g_i2c1_handle.instance->CR2 &= ~I2C_CR2_DMAEN; /* 关闭 DMA 请求 */

    /* 等待最后一个字节完全脱离总线 (BTF 置位) */
    while (!(g_i2c1_handle.instance->SR1 & I2C_SR1_BTF))
    {
    }

    g_i2c1_handle.instance->CR1 |= I2C_CR1_STOP;   /* 产生 STOP 释放总线 */
    g_i2c1_handle.state = I2C_STATE_IDLE;
    /* I2C_DmaTxCpltCallback(&g_i2c1_handle);  释放信号量唤醒线程 */
}

/* DMA 接收完成中断 */
void DMA1_Channel_I2C_RX_IRQHandler(void)
{
    DMA1->IFCR |= (1U << 5);                       /* 清除 DMA 完成标志 */
    g_i2c1_handle.instance->CR2 &= ~(I2C_CR2_DMAEN | I2C_CR2_LAST);

    g_i2c1_handle.instance->CR1 |= I2C_CR1_STOP;   /* 产生 STOP 释放总线 */
    g_i2c1_handle.state = I2C_STATE_IDLE;
    /* I2C_DmaRxCpltCallback(&g_i2c1_handle); */
}
```

> [!WARNING] DMA 发送完成 ≠ 物理发送完成
> DMA TCIF 只代表数据搬进了 DR，最后一字节可能仍在移位。发送完成中断里必须**等 BTF 置位**再产生 STOP，否则会截断尾字节——与 [[modbus]] 的 RS-485 DE 切换是同一类时序陷阱。

---

## 🆚 5. 四种方式对比

| 方式 | CPU 占用 | 实现复杂度 | 数据量 | 阻塞性 | 典型用途 |
| :--- | :--: | :--: | :--: | :--: | :--- |
| **软件模拟 I²C** | 很高 | 中 | 很低 | 阻塞 | 无 I²C 外设、GPIO 复用、DEBUG |
| **硬件轮询** | 高 | 低 | 低 | 阻塞 | 少量寄存器读写、初始化 |
| **硬件中断** | 中低 | 中 | 中 | 非阻塞 | 普通传感器驱动 |
| **I²C + DMA** | 最低 | 高 | 大 | 非阻塞 | EEPROM、大批量数据搬运 |

---

## 🔗 思考与延伸索引

- [[I2C]] — I²C 协议层总览：帧格式、仲裁、Clock Stretching
- [[SPI-code]] — 与 I²C 实现方式高度相似的 SPI 驱动四态
- [[UART-code]] — UART 驱动的轮询/中断/DMA 实现对照
- [[DMA、Buffer 与流量控制]] — DMA TC 语义、环形缓冲与背压
- [[嵌入式内核]] — 中断下半部、状态机与并发保护

---

## 📚 参考资料

1. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（I2C 事件序列 / DMA 章节）.
2. **STMicroelectronics** — *AN2824: I2C optimized firmware examples*.
3. **NXP Semiconductors** — *The I²C-bus specification, UM10204*.
4. 项目内部笔记：[[I2C]]、[[DMA、Buffer 与流量控制]]。
