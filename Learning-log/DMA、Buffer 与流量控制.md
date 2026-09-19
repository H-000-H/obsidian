# DMA、Buffer 与流量控制（背压）

## 1. DMA 的本质

DMA（Direct Memory Access）本质上只是：

> **在外设与内存之间自动搬运数据，减少 CPU 参与数据搬运的开销。**

典型数据流：

```text
外设 → DMA → RAM
RAM → DMA → 外设
```

DMA 本身通常不负责：

- 协议解析
    
- 数据包边界判断
    
- CRC / 帧格式解析
    
- 上层业务处理
    
- 判断“这是不是完整的一帧”
    

因此：

```text
DMA负责搬
CPU/Parser负责理解
```

---

# 2. DMA TC 到底表示什么

DMA 的 `TC（Transfer Complete）` 表示：

> **本次配置的 DMA 传输数量已经搬完。**

例如：

```text
DMA配置：
Buffer = 1024 B
Transfer Count = 1024
```

DMA TC：

```text
DMA
 ↓
搬完 1024 B
 ↓
TC
```

并不意味着：

> “整个通信协议事务结束了”。

必须结合具体外设判断。

例如：

|外设|DMA TC 的含义|真正事务边界可能由什么决定|
|---|---|---|
|UART RX|配置数量的数据搬完|IDLE、长度、协议帧|
|UART TX|数据已经搬到外设|TX Complete / TC / BSY|
|SPI|配置的数据搬完|CS/NSS、传输长度|
|I²C|配置的数据搬完|STOP、地址阶段、读写长度等|

---

# 3. TX 和 RX 的生产者/消费者关系

理解 DMA 最重要的是先确定数据方向。

## 3.1 TX：MCU → 外部设备

例如 LCD：

```text
LVGL
  ↓
Display Buffer
  ↓
SPI DMA
  ↓
SPI
  ↓
LCD
```

这里：

```text
LVGL = Producer
DMA/SPI = Consumer
```

如果：

```text
LVGL产生速度 > SPI发送速度
```

主要表现为：

```text
LVGL
 ↓
等待 / 排队
 ↓
SPI
 ↓
LCD
```

最终：

> **刷新吞吐量被 SPI 带宽限制，FPS 上不去。**

通常不会因为 LVGL 产生得快就无限制地把数据“冲进 DMA”。

可以利用 DMA 完成回调形成同步：

```text
LVGL提交 Buffer A
       ↓
SPI DMA发送
       ↓
TC
       ↓
flush_ready
       ↓
LVGL可以继续使用/提交
```

因此可以把发送速度锁在 DMA/SPI 能承受的范围内。

---

# 4. RX：外部设备 → MCU

接收方向则相反：

```text
外部设备
   ↓
DMA
   ↓
Buffer
   ↓
CPU / Parser
```

这里：

```text
外部设备 = Producer
CPU/Parser = Consumer
```

如果：

```text
外部产生速度 > CPU处理速度
```

数据就会不断积累：

```text
外部设备
████████████████████

DMA
████████████████████

CPU
███████
```

最终：

```text
Buffer越来越满
      ↓
Buffer满
      ↓
新数据继续到达
      ↓
覆盖 / FIFO Overflow
      ↓
丢数据
```

---

# 5. 为什么 RX 不能简单靠增大 Buffer 解决

假设：

```text
外部设备：10 MB/s
CPU解析：  3 MB/s
```

那么：

```text
Buffer积累速度
= 10 - 3
= 7 MB/s
```

无论 Buffer 是：

```text
4 KB
4 MB
40 MB
```

只要：

```text
生产速度 > 消费速度
```

最终都会满。

因此：

> **扩大 Buffer 只能延缓溢出，不能解决长期吞吐量不匹配。**

---

# 6. Circular DMA 与 Ring Buffer

## 6.1 Circular DMA

普通数组：

```c
uint8_t rx_buf[4096];
```

完全可以配置成 Circular DMA。

硬件行为：

```text
0 ───────────────── 4095
↑                     │
└─────────────────────┘
```

DMA 到末尾后重新从 0 开始写。

---

## 6.2 软件 Ring 管理

通常软件维护两个位置：

```c
size_t dma_pos;   // DMA当前写位置
size_t read_pos;  // CPU已经消费的位置
```

数据流：

```text
                 DMA写
                   ↓
┌──────────────────────────┐
│                          │
└──────────────────────────┘
↑                          │
└──────────────────────────┘
↑
CPU读
```

所以：

```text
DMA写指针 + CPU读指针
        ↓
软件Ring Buffer
```

注意：

> Circular DMA 是硬件的循环写入机制；Ring Buffer 是软件对这块内存的管理方式。

---

# 7. Normal DMA 与线性 Buffer

也可以不用 Circular DMA。

例如：

```text
Buffer A
0 ───────────────→ END
                     ↓
                    TC
                     ↓
                 处理数据
                     ↓
                 重新配置 DMA
```

然后：

```text
Buffer B
0 ───────────────→ END
                     ↓
                    TC
```

也可以使用：

```text
Buffer A ↔ Buffer B
```

形成 Ping-Pong / Double Buffer：

```text
DMA → Buffer B
CPU → Buffer A

DMA → Buffer A
CPU → Buffer B
```

这样 DMA 和 CPU 可以并行工作。

---

# 8. Circular DMA 也不是“永不丢数据”

Circular DMA 只是：

> **DMA 到 Buffer 尾部后自动回到头部继续写。**

它不能增加 Buffer 容量。

如果：

```text
DMA写速度 > CPU消费速度
```

最终：

```text
DMA绕回
 ↓
写到CPU还没消费的位置
 ↓
覆盖旧数据
 ↓
丢数据
```

所以：

```text
Circular DMA
≠ 无限缓存
```

---

# 9. 真正解决 RX 溢出的办法：背压

如果：

```text
Producer > Consumer
```

真正有效的解决方向之一是：

> **让 Producer 暂停或降低发送速度。**

这就是：

# Backpressure（背压）

基本模型：

```text
Producer
   ↓
 Buffer
   ↓
Consumer
```

当 Buffer 快满：

```text
Consumer处理不过来
        ↓
Buffer达到高水位
        ↓
要求Producer暂停
        ↓
Buffer下降
        ↓
允许Producer继续
```

---

# 10. 高水位 / 低水位

不能等 Buffer 100% 满了才发送暂停信号。

例如：

```text
Buffer = 4096 B
```

可以设置：

```text
80% → PAUSE
40% → RESUME
```

模型：

```text
100% ─────────────── Overflow
 │
 │
80%  ─────────────── PAUSE
 │
 │    安全余量
 │
40%  ─────────────── RESUME
 │
0%   ─────────────── Empty
```

为什么不能等满了才 PAUSE？

因为：

```text
Buffer达到高水位
      ↓
发送PAUSE
      ↓
信号传输
      ↓
对端收到
      ↓
对端停止发送
```

这个过程中数据仍可能继续到达。

因此必须预留：

> **流控延迟期间的安全空间。**

---

# 11. 如果对端支持流控

例如：

```text
MCU
 ↓
PAUSE
 ↓
上位机
 ↓
停止发送
```

之后：

```text
CPU消费Buffer
 ↓
Buffer下降
 ↓
RESUME
 ↓
上位机继续发送
```

这样就形成完整的背压。

---

# 12. 如果对端不支持流控

例如 MCU 发送：

```text
PAUSE
```

但上位机完全不认识：

```text
上位机
████████████████████
        ↓
       MCU
        ↓
“我不接收了”
```

上位机仍然继续发送。

最终：

```text
FIFO满
   ↓
Overflow

或者：

Ring Buffer被覆盖
   ↓
丢数据
```

所以：

> **流控必须是通信双方都遵守的机制。**

单方面说“我不接收了”，不能阻止对方继续发送。

---

# 13. ISR 在流控中的作用

通常不是：

```text
ISR
 ↓
做大量协议解析
```

而是：

```text
ISR
 ↓
检测Buffer水位
 ↓
设置状态
 ↓
触发流控
 ↓
通知任务
```

例如：

```text
Buffer >= HIGH_WATERMARK
        ↓
ISR设置：
flow_pause = true
        ↓
RTS / XOFF / 协议控制
        ↓
通知发送端暂停
```

之后：

```text
Task
 ↓
消费Buffer
 ↓
Buffer <= LOW_WATERMARK
 ↓
flow_pause = false
 ↓
允许继续发送
```

---

# 14. UART RX

UART 通常是比较典型的被动接收：

```text
外部设备
    ↓
  UART
    ↓
  DMA
    ↓
RX Buffer
    ↓
  Parser
```

UART 没有 SPI/I²C 那种由接收方控制数据时钟的机制。

因此如果：

```text
外部发送速度 > MCU处理速度
```

需要考虑：

- RTS/CTS 硬件流控
    
- XON/XOFF 软件流控
    
- ACK / Window
    
- 协议层限速
    
- 增大 Buffer
    
- 提高 Parser 消费速度
    

否则最终可能丢数据。

---

# 15. SPI Master RX

SPI Master 与 UART 最大的区别之一：

> **Master 控制 SCK。**

例如 MCU 读取 SPI 从设备：

```text
MCU
 │
 ├── SCK ─────→ Slave
 │
 └←── DATA ───── Slave
```

没有 SCK：

```text
Slave
 ↓
无法继续输出下一位数据
```

所以 Master 可以控制接收节奏。

例如：

```text
读1 KB
 ↓
DMA
 ↓
TC
 ↓
停止SCK
 ↓
CPU处理
 ↓
继续产生SCK
 ↓
再读1 KB
```

因此 SPI Master RX 可以天然形成一种背压。

---

# 16. SPI Slave RX

如果 MCU 是 SPI Slave：

```text
外部 Master
     ↓ SCK
     ↓ DATA
    MCU
     ↓
    DMA
     ↓
  Buffer
```

这时：

> **SCK 是外部 Master 控制的。**

如果 Master 不管 MCU 是否处理得过来，继续提供 Clock：

```text
Master速度 > CPU消费速度
        ↓
Buffer积累
        ↓
Buffer满
        ↓
Overflow / 覆盖
```

这时需要额外的：

- READY GPIO
    
- 协议层 ACK
    
- Window / Credit
    
- Master 查询状态
    

等流控机制。

---

# 17. I²C Master RX

I²C Master 和 SPI Master 类似：

```text
Master控制 SCL
```

读取：

```text
Master
  │
  │ SCL
  ↓
Slave
  │
  │ SDA
  ↓
Master
```

Master 不继续产生 SCL：

```text
Slave
 ↓
无法继续输出后续数据
```

因此 Master 可以：

```text
读一块
 ↓
停止/结束事务
 ↓
CPU处理
 ↓
再发起下一块读取
```

这是一种天然的节流方式。

---

# 18. I²C Slave 与 Clock Stretching

I²C 从设备还有一个特殊机制：

> **Clock Stretching**

Slave 可以把 SCL 拉低：

```text
Master                  Slave
  │                       │
  │──── SCL/SDA ─────────→│
  │                       │
  │                  Buffer快满
  │                       │
  │←──── SCL拉低 ─────────│
  │                       │
  │       暂停时钟         │
  │                       │
  │                  CPU处理数据
  │                  Buffer下降
  │                       │
  │←──── 释放SCL ─────────│
  │                       │
  │──── 继续SCL ─────────→│
```

所以：

> **I²C Slave 可以通过 Clock Stretching 暂停当前正在进行的 I²C 事务。**

这相当于一种硬件级背压。

但要注意：

```text
Clock Stretching
        ↓
暂停当前事务
```

它不是：

```text
禁止Master以后发起任何新的事务
```

下一笔事务仍然由 Master 发起。

---

# 19. SPI / I²C / UART 的核心区别

|场景|谁控制数据产生节奏|接收端能否直接控制节奏|
|---|---|---|
|UART RX|外部设备|通常需要额外流控|
|SPI Master RX|MCU Master|可以控制 SCK|
|SPI Slave RX|外部 Master|需要额外流控|
|I²C Master RX|MCU Master|可以控制 SCL|
|I²C Slave RX|Master，但 Slave可 Clock Stretch|可以暂停当前事务|

因此：

> **判断 RX 是否容易产生溢出，关键不是“有没有 DMA”，而是“接收端能不能控制 Producer 的产生速度”。**

---

# 20. DMA Buffer 的两种典型管理方式

## Circular DMA

```text
外部设备
    ↓
   DMA
    ↓
┌──────────────────────┐
│      Ring Buffer     │
└──────────────────────┘
   ↑              ↑
read_pos       dma_pos
   │              │
   └──→ Parser ←──┘
```

特点：

- DMA 可以持续运行
    
- Buffer 到尾部自动回绕
    
- CPU 根据 DMA 写位置寻找新数据
    
- 适合连续流
    
- 必须防止 DMA 覆盖 CPU 尚未消费的数据
    

---

## Normal DMA

```text
外部设备
    ↓
   DMA
    ↓
┌──────────────┐
│   Buffer A   │
└──────────────┘
       ↓ TC
      CPU处理
       ↓
重新配置DMA
       ↓
┌──────────────┐
│   Buffer B   │
└──────────────┘
```

特点：

- 一次处理一段数据
    
- TC 后重新配置
    
- 可以结合 Ping-Pong / Double Buffer
    
- 适合分块传输
    

---

# 21. 最终统一模型

无论 UART、SPI、I²C，先看：

```text
Producer
    ↓
通信外设
    ↓
DMA
    ↓
Buffer
    ↓
CPU / Parser
    ↓
Consumer
```

如果：

```text
Producer ≤ Consumer
```

```text
正常
```

如果短时间：

```text
Producer > Consumer
```

```text
Buffer吸收突发
```

如果长期：

```text
Producer > Consumer
```

那么：

```text
Buffer最终一定会满
```

这时候必须至少做到以下之一：

```text
1. 降低 Producer 速度
2. 让 Producer 暂停（Backpressure）
3. 提高 Consumer 处理速度
4. 增加缓存，只能延缓溢出
5. 如果协议允许，允许丢弃数据
```

最关键的一条：

> **DMA 解决的是“搬运效率”，Buffer 解决的是“短期速率不匹配”，背压解决的是“控制生产速度”。DMA、Ring Buffer、Double Buffer 本身都不能让一个长期生产速度高于消费速度的系统凭空变得不溢出。**