 1. 软件轮询模拟I2C
 
 ```
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
# 1. 初始化
# ----------------------------------------------------
def I2cInit():
    # 空闲状态：SDA 与 SCL 均由上拉电阻拉高
    GpioSdaHigh()
    GpioSclHigh()
    I2cDelay()

# ----------------------------------------------------
# 2. 起始信号 (START)
# 条件：SCL 为高时，SDA 由高变低
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
# 3. 重复起始信号 (RESTART)
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
# 4. 停止信号 (STOP)
# 条件：SCL 为高时，SDA 由低变高
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
        
        # 产生 SCL 上升沿并保持高，让从机采样
        GpioSclHigh()
        I2cDelay()
        
        # 拉低 SCL，准备下一 bit
        GpioSclLow()
        I2cDelay()

    # 2. 接收从机应答（第 9 个时钟周期）
    # 主机释放 SDA（拉高），让从机控制 SDA
    GpioSdaHigh()
    I2cDelay()
    
    GpioSclHigh()       # SCL 拉高供主机采样 ACK
    I2cDelay()
    ack = GpioSdaRead() # 读取从机应答：0 为 ACK，1 为 NACK
    
    GpioSclLow()        # 结束第 9 个脉冲
    I2cDelay()
    
    return (ack == 0)

# ----------------------------------------------------
# 6. 读取一个字节 (Read Byte) + 发送应答
# send_ack: True 表示回复从机 ACK(继续读)，False 表示 NACK(停止读)
# ----------------------------------------------------
def I2cReadByte(send_ack: bool) -> int:
    byte_data = 0
    # 主机释放 SDA，准备读取
    GpioSdaHigh()

    # 1. 串行读取 8 位
    for i in range(8):
        byte_data <<= 1
        GpioSclLow()
        I2cDelay()
        
        GpioSclHigh()   # SCL 拉高，此时从机已准备好数据
        I2cDelay()
        
        if GpioSdaRead():
            byte_data |= 0x01
    
    GpioSclLow()
    I2cDelay()

    # 2. 主机发送应答给从机（第 9 个时钟周期）
    if send_ack:
        GpioSdaLow()    # 发送 ACK (0)
    else:
        GpioSdaHigh()   # 发送 NACK (1)
    
    I2cDelay()
    GpioSclHigh()       # SCL 脉冲，让从机读取应答
    I2cDelay()
    GpioSclLow()
    I2cDelay()
    
    GpioSdaHigh()       # 释放 SDA
    return byte_data

 ```
 2.硬件I2C
 ```
 #include <stdint.h>
#include <stdbool.h>

/* 硬件 I2C 寄存器结构体映射 (以常见硬件外设为例) */
typedef struct {
    volatile uint32_t CR1;      /* 控制寄存器 1 (使能、START、STOP、ACK控制) */
    volatile uint32_t CR2;      /* 控制寄存器 2 (时钟频率等) */
    volatile uint32_t OAR1;     /* 自身地址寄存器 */
    volatile uint32_t DR;       /* 数据寄存器 (Tx/Rx) */
    volatile uint32_t SR1;      /* 状态寄存器 1 (事件标志位) */
    volatile uint32_t SR2;      /* 状态寄存器 2 (系统状态标志位) */
} Hardware_I2C_TypeDef;

/* 状态寄存器 SR1 常用标志位掩码 */
#define I2C_SR1_SB      (1U << 0)   /* Start Bit: 起始位已发送 */
#define I2C_SR1_ADDR    (1U << 1)   /* Address Sent: 从机地址发送完成且收到 ACK */
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
 * @param dev_addr  7位从机设备地址
 * @param data      待发送数据缓冲区
 * @param len       数据长度
 * @return true 写入成功, false 超时或无应答 (NACK)
 */
bool I2C_Master_Transmit_Poll(Hardware_I2C_TypeDef *i2c, uint8_t dev_addr, const uint8_t *data, uint16_t len)
{
    uint32_t timeout;

    /* 1. 等待总线空闲 (Bus not busy) */
    timeout = 100000;
    while ((i2c->SR2 & I2C_SR2_BUSY) && (--timeout));
    if (timeout == 0) return false;

    /* 2. 硬件产生 START 信号 */
    i2c->CR1 |= I2C_CR1_START;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_SB) && (--timeout));  /* 轮询 SB 标志位置位 */
    if (timeout == 0) return false;

    /* 3. 发送 7位地址 + 写方向位 (0) */
    i2c->DR = (uint8_t)(dev_addr << 1) | 0x00;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_ADDR) && (--timeout)) /* 轮询从机是否回复 ACK */
    {
        if (i2c->SR1 & I2C_SR1_AF) {                  /* 检查是否收到 NACK */
            i2c->CR1 |= I2C_CR1_STOP;
            i2c->SR1 &= ~I2C_SR1_AF;                  /* 清除失败标志 */
            return false;
        }
    }
    if (timeout == 0) return false;

    /* 清除 ADDR 标志位 (硬件标准流程：先读 SR1，再读 SR2) */
    (void)i2c->SR1;
    (void)i2c->SR2;

    /* 4. 轮询发送数据载荷 */
    for (uint16_t i = 0; i < len; ++i)
    {
        timeout = 100000;
        while (!(i2c->SR1 & I2C_SR1_TXE) && (--timeout)); /* 等待发送缓冲区空 */
        if (timeout == 0) return false;

        i2c->DR = data[i]; /* 写入字节到硬件移位器 */
    }

    /* 5. 等待最后一个字节完全从总线移出 (Byte Transfer Finished) */
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_BTF) && (--timeout));
    if (timeout == 0) return false;

    /* 6. 产生 STOP 信号释放总线 */
    i2c->CR1 |= I2C_CR1_STOP;

    return true;
}

/**
 * @brief 硬件 I2C 阻塞轮询读取
 * @param i2c       外设基地址
 * @param dev_addr  7位从机设备地址
 * @param buf       接收缓冲区
 * @param len       需要读取的字节数
 * @return true 读取成功, false 失败
 */
bool I2C_Master_Receive_Poll(Hardware_I2C_TypeDef *i2c, uint8_t dev_addr, uint8_t *buf, uint16_t len)
{
    uint32_t timeout;
    if (len == 0) return false;

    /* 1. 等待总线空闲 */
    timeout = 100000;
    while ((i2c->SR2 & I2C_SR2_BUSY) && (--timeout));
    if (timeout == 0) return false;

    /* 使能硬件自动回复 ACK */
    i2c->CR1 |= I2C_CR1_ACK;

    /* 2. 产生 START */
    i2c->CR1 |= I2C_CR1_START;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_SB) && (--timeout));
    if (timeout == 0) return false;

    /* 3. 发送 7位地址 + 读方向位 (1) */
    i2c->DR = (uint8_t)(dev_addr << 1) | 0x01;
    timeout = 100000;
    while (!(i2c->SR1 & I2C_SR1_ADDR) && (--timeout));
    if (timeout == 0) return false;

    /* 特殊边界：如果只读 1 个字节 */
    if (len == 1)
    {
        /* 硬件规定：单字节读取需在清除 ADDR 前先关闭 ACK，并请求 STOP */
        i2c->CR1 &= ~I2C_CR1_ACK;
        (void)i2c->SR1;
        (void)i2c->SR2;
        i2c->CR1 |= I2C_CR1_STOP;

        /* 等待数据落入 RXNE */
        timeout = 100000;
        while (!(i2c->SR1 & I2C_SR1_RXNE) && (--timeout));
        if (timeout == 0) return false;

        buf[0] = (uint8_t)i2c->DR;
        return true;
    }

    /* 多字节：清除 ADDR 标志进入接收 */
    (void)i2c->SR1;
    (void)i2c->SR2;

    /* 4. 循环读取 */
    for (uint16_t i = 0; i < len; ++i)
    {
        /* 倒数第 1 个字节时，必须提前置 NACK 并发 STOP 信号 */
        if (i == len - 1)
        {
            i2c->CR1 &= ~I2C_CR1_ACK;
            i2c->CR1 |= I2C_CR1_STOP;
        }

        /* 轮询等待数据接收完成标志 (RXNE) */
        timeout = 100000;
        while (!(i2c->SR1 & I2C_SR1_RXNE) && (--timeout));
        if (timeout == 0) return false;

        buf[i] = (uint8_t)i2c->DR;
    }

    return true;
}

 ```
3. 中断硬件I2C
```
#include <stdint.h>
#include <stdbool.h>

typedef enum {
    I2C_STATE_IDLE = 0,
    I2C_STATE_BUSY_TX,
    I2C_STATE_BUSY_RX,
    I2C_STATE_COMPLETE,
    I2C_STATE_ERROR
} I2C_State_t;

typedef struct {
    Hardware_I2C_TypeDef *instance; // I2C 外设基地址
    uint8_t               dev_addr; // 7位设备地址
    uint8_t              *buffer;   // 数据缓冲区指针
    volatile uint16_t     length;   // 总字节数
    volatile uint16_t     index;    // 当前已传输索引
    volatile I2C_State_t  state;    // 传输状态机
    volatile uint32_t     error_code;
} I2C_Handle_t;

static I2C_Handle_t g_i2c1_handle;

// 中断使能控制位 (CR2 寄存器)
#define I2C_CR2_ITEVTEN   (1U << 9)   // 事件中断使能
#define I2C_CR2_ITBUFEN   (1U << 10)  // 缓冲中断使能 (TXE/RXNE)
#define I2C_CR2_ITERREN   (1U << 8)   // 错误中断使能

/**
 * @brief 中断模式启动数据发送 (非阻塞)
 */
bool I2C_Master_Transmit_IT(I2C_Handle_t *hi2c, uint8_t dev_addr, const uint8_t *data, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0) return false;

    hi2c->dev_addr = dev_addr;
    hi2c->buffer   = (uint8_t *)data;
    hi2c->length   = len;
    hi2c->index    = 0;
    hi2c->state    = I2C_STATE_BUSY_TX;

    // 开启事件中断、缓冲中断、错误中断
    hi2c->instance->CR2 |= (I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);

    // 硬件发送 START 信号，触发第一次 EV 中断 (SB 标志)
    hi2c->instance->CR1 |= I2C_CR1_START;

    return true;
}

/**
 * @brief 中断模式启动数据接收 (非阻塞)
 */
bool I2C_Master_Receive_IT(I2C_Handle_t *hi2c, uint8_t dev_addr, uint8_t *buf, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0) return false;

    hi2c->dev_addr = dev_addr;
    hi2c->buffer   = buf;
    hi2c->length   = len;
    hi2c->index    = 0;
    hi2c->state    = I2C_STATE_BUSY_RX;

    // 开启应答 ACK
    hi2c->instance->CR1 |= I2C_CR1_ACK;
    hi2c->instance->CR2 |= (I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);

    // 发送 START 信号
    hi2c->instance->CR1 |= I2C_CR1_START;

    return true;
}

void I2C1_EV_IRQHandler(void)
{
    I2C_Handle_t *hi2c = &g_i2c1_handle;
    uint32_t sr1 = hi2c->instance->SR1;
    uint32_t sr2 = hi2c->instance->SR2;

    /* ---------------- 1. SB: 起始条件产生 ---------------- */
    if (sr1 & I2C_SR1_SB)
    {
        // 写模式传 (addr<<1)|0，读模式传 (addr<<1)|1
        uint8_t rw_bit = (hi2c->state == I2C_STATE_BUSY_RX) ? 0x01 : 0x00;
        hi2c->instance->DR = (uint8_t)(hi2c->dev_addr << 1) | rw_bit;
        return; // 写 DR 自动清除 SB
    }

    /* ---------------- 2. ADDR: 地址发送完毕且收到从机 ACK ---------------- */
    if (sr1 & I2C_SR1_ADDR)
    {
        if (hi2c->state == I2C_STATE_BUSY_RX)
        {
            // 单字节接收特殊处理：在清 ADDR 前关闭 ACK 并请求 STOP
            if (hi2c->length == 1)
            {
                hi2c->instance->CR1 &= ~I2C_CR1_ACK;
                hi2c->instance->CR1 |= I2C_CR1_STOP;
            }
        }
        // 软件读 SR1 再读 SR2，即可清除 ADDR 标志
        (void)sr1;
        (void)sr2;
        return;
    }

    /* ---------------- 3. TXE: 发送数据缓冲区空 (写数据流) ---------------- */
    if ((sr1 & I2C_SR1_TXE) && (hi2c->state == I2C_STATE_BUSY_TX))
    {
        if (hi2c->index < hi2c->length)
        {
            hi2c->instance->DR = hi2c->buffer[hi2c->index++];
        }
        else
        {
            // 数据已全部送入移位寄存器，关闭 ITBUFEN 防止反复触发 TXE 中断
            // 剩下的交给 BTF 中断去确认最后字节完全飞出总线
            hi2c->instance->CR2 &= ~I2C_CR2_ITBUFEN;
        }
        return;
    }

    /* ---------------- 4. BTF: 字节传输完全结束 (发送尾部) ---------------- */
    if ((sr1 & I2C_SR1_BTF) && (hi2c->state == I2C_STATE_BUSY_TX))
    {
        if (hi2c->index >= hi2c->length)
        {
            // 产生 STOP 信号释放总线
            hi2c->instance->CR1 |= I2C_CR1_STOP;

            // 关闭所有中断
            hi2c->instance->CR2 &= ~(I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
            hi2c->state = I2C_STATE_COMPLETE;

            // 回调用户完成函数 (或者发信号量给应用任务)
            // I2C_TxCpltCallback(hi2c);
        }
        return;
    }

    /* ---------------- 5. RXNE: 接收数据寄存器非空 (读数据流) ---------------- */
    if ((sr1 & I2C_SR1_RXNE) && (hi2c->state == I2C_STATE_BUSY_RX))
    {
        hi2c->buffer[hi2c->index++] = (uint8_t)hi2c->instance->DR;

        // 倒数第 1 个字节到来前，提前关闭 ACK 并发送 STOP
        if (hi2c->index == (hi2c->length - 1))
        {
            hi2c->instance->CR1 &= ~I2C_CR1_ACK;
            hi2c->instance->CR1 |= I2C_CR1_STOP;
        }
        else if (hi2c->index >= hi2c->length)
        {
            // 全部接收完成，关闭中断
            hi2c->instance->CR2 &= ~(I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
            hi2c->state = I2C_STATE_COMPLETE;

            // 回调用户接收完成函数
            // I2C_RxCpltCallback(hi2c);
        }
        return;
    }
}
void I2C1_ER_IRQHandler(void)
{
    I2C_Handle_t *hi2c = &g_i2c1_handle;
    uint32_t sr1 = hi2c->instance->SR1;

    // AF (Acknowledge Failure): 从机没有应答 (NACK)
    if (sr1 & I2C_SR1_AF)
    {
        hi2c->instance->CR1 |= I2C_CR1_STOP; // 释放总线
        hi2c->instance->SR1 &= ~I2C_SR1_AF;  // 写 0 清除标志位
        hi2c->error_code |= 0x04;
    }

    // BERR (Bus Error): 总线在错误的时间出现了 START/STOP 沿
    if (sr1 & (1U << 8))
    {
        hi2c->instance->SR1 &= ~(1U << 8);
        hi2c->error_code |= 0x01;
    }

    // 关闭中断，将状态设为错误
    hi2c->instance->CR2 &= ~(I2C_CR2_ITEVTEN | I2C_CR2_ITBUFEN | I2C_CR2_ITERREN);
    hi2c->state = I2C_STATE_ERROR;

    // I2C_ErrorCallback(hi2c);
}

```
  4. DMA+I2C+中断
```
#include <stdint.h>
#include <stdbool.h>

/* 控制寄存器 CR2 的 DMA 请求使能位 */
#define I2C_CR2_DMAEN     (1U << 11)  /* DMA 请求使能：只要 TXE/RXNE 置位，外设向 DMA 控制器发硬件请求 */
#define I2C_CR2_LAST      (1U << 12)  /* 接收模式下的特殊位：指示下一次 DMA 传输是最后一个字节 (产生 NACK) */

/* DMA 通道寄存器结构体 (概念映射) */
typedef struct {
    volatile uint32_t CCR;    /* 控制寄存器 (使能、传输方向、自增、中断使能) */
    volatile uint32_t CNDTR;  /* 传输计数器 (字节数，每发一个减1) */
    volatile uint32_t CPAR;   /* 外设地址 (&i2c->DR) */
    volatile uint32_t CMAR;   /* 内存地址 (SRAM 缓冲区) */
} Hardware_DMA_Channel_TypeDef;
/**
 * @brief 启动 I2C DMA 主机发送
 */
bool I2C_Master_Transmit_DMA(I2C_Handle_t *hi2c, Hardware_DMA_Channel_TypeDef *dma_tx,
                             uint8_t dev_addr, const uint8_t *data, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0) return false;

    hi2c->state = I2C_STATE_BUSY_TX;
    hi2c->dev_addr = dev_addr;

    /* 1. 配置 DMA 通道 */
    dma_tx->CCR &= ~1U;                       /* 关闭 DMA 通道准备配置 */
    dma_tx->CPAR = (uint32_t)&hi2c->instance->DR; /* 外设目标：I2C 数据寄存器 */
    dma_tx->CMAR = (uint32_t)data;            /* 内存源：数据首地址 */
    dma_tx->CNDTR = len;                      /* 传输长度 */
    dma_tx->CCR |= (1U << 4);                 /* 方向：Memory -> Peripheral */
    dma_tx->CCR |= (1U << 7);                 /* 内存地址自增 */
    dma_tx->CCR |= (1U << 1);                 /* 开启 DMA 传输完成中断 (TCIE) */
    dma_tx->CCR |= 1U;                        /* 使能 DMA 通道 */

    /* 2. 使能 I2C 的 DMA 请求 */
    hi2c->instance->CR2 |= I2C_CR2_DMAEN;

    /* 3. 硬件产生 START 信号 (必须先发 START 和寻址) */
    hi2c->instance->CR1 |= I2C_CR1_START;

    /* 4. 在 EV 事件中断里发完地址后，DMA 会立刻接管后续所有数据搬运 */
    // 开启事件中断，只用于处理 SB 和 ADDR 两个阶段
    hi2c->instance->CR2 |= I2C_CR2_ITEVTEN;

    return true;
}
/**
 * @brief 启动 I2C DMA 主机接收
 */
bool I2C_Master_Receive_DMA(I2C_Handle_t *hi2c, Hardware_DMA_Channel_TypeDef *dma_rx,
                            uint8_t dev_addr, uint8_t *buf, uint16_t len)
{
    if (hi2c->state != I2C_STATE_IDLE || len == 0) return false;

    hi2c->state = I2C_STATE_BUSY_RX;
    hi2c->dev_addr = dev_addr;

    /* 1. 配置 DMA RX 通道 */
    dma_rx->CCR &= ~1U;
    dma_rx->CPAR = (uint32_t)&hi2c->instance->DR;
    dma_rx->CMAR = (uint32_t)buf;
    dma_rx->CNDTR = len;
    dma_rx->CCR &= ~(1U << 4);                /* 方向：Peripheral -> Memory */
    dma_rx->CCR |= (1U << 7);                 /* 内存地址自增 */
    dma_rx->CCR |= (1U << 1);                 /* 开启 DMA 完成中断 (TCIE) */
    dma_rx->CCR |= 1U;                        /* 使能 DMA 通道 */

    /* 2. 使能 ACK 并开启 DMA 请求 */
    hi2c->instance->CR1 |= I2C_CR1_ACK;
    hi2c->instance->CR2 |= I2C_CR2_DMAEN;

    /* 
     * 关键配置：使能 LAST 位
     * 作用：当 DMA 将 CNDTR 倒数到最后 1 次传输时，I2C 硬件自动将 ACK 置为 0 (发送 NACK)
     */
    if (len > 1) {
        hi2c->instance->CR2 |= I2C_CR2_LAST;
    }

    /* 3. 产生 START */
    hi2c->instance->CR1 |= I2C_CR1_START;
    hi2c->instance->CR2 |= I2C_CR2_ITEVTEN;   /* 进入中断处理 SB 和 ADDR */

    return true;
}
/* DMA 发送完成中断 */
void DMA1_Channel_I2C_TX_IRQHandler(void)
{
    // 清除 DMA 传输完成标志 (TCIF)
    DMA1->IFCR |= (1U << 1); 

    // 关闭 DMA 请求
    g_i2c1_handle.instance->CR2 &= ~I2C_CR2_DMAEN;

    /* 等待最后一个字节完全脱离总线 (BTF 置位) */
    while (!(g_i2c1_handle.instance->SR1 & I2C_SR1_BTF));

    /* 产生 STOP 停止条件释放总线 */
    g_i2c1_handle.instance->CR1 |= I2C_CR1_STOP;

    g_i2c1_handle.state = I2C_STATE_IDLE;

    // 回调通知上层完成：如释放信号量唤醒 UI / 传感器线程
    // I2C_DmaTxCpltCallback(&g_i2c1_handle);
}

/* DMA 接收完成中断 */
void DMA1_Channel_I2C_RX_IRQHandler(void)
{
    // 清除 DMA 完成标志
    DMA1->IFCR |= (1U << 5);

    // 关闭 DMA 请求与 LAST 控制
    g_i2c1_handle.instance->CR2 &= ~(I2C_CR2_DMAEN | I2C_CR2_LAST);

    /* 产生 STOP 释放总线 */
    g_i2c1_handle.instance->CR1 |= I2C_CR1_STOP;

    g_i2c1_handle.state = I2C_STATE_IDLE;

    // 回调通知上层
    // I2C_DmaRxCpltCallback(&g_i2c1_handle);
}

```