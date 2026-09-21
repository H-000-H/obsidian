---
title: RS-485 Modbus RTU 驱动与时序完整技术手册
tags:
  - Embedded
  - Protocol
  - ModbusRTU
  - RS485
  - Driver
  - STM32
  - Timing
standard: "Modbus over Serial Line Specification and Implementation Guide V1.02"
status: Completed
created: 2026-09-20
---

# 📝 RS-485 Modbus RTU 驱动与时序完整技术手册

> [!INFO] 笔记概览
> - **参考标准**：Modbus over Serial Line Specification and Implementation Guide V1.02
> - **内容范围**：官方时序公式 · 临界波特率断点 · DE/RE 切换 · RX 状态机 · CRC-16 校验 · STM32 C 语言参考实现
> - **推荐路径**：§1 时序 → §2 TX → §3 RX 状态机 → §4 CRC 校验 → §5 代码 → §6 排查清单

---

## ⏱️ 1. Modbus RTU 官方时序规范与数学计算

Modbus RTU 依靠物理总线上的**时间静默（Silence / Gap）**作为帧起始与结束的唯一定界符，任何字符间"停顿"过长都会被视为一帧结束。

```text
总线状态: ────[ ≥ 3.5 字符静默 ]───┬──[ 报文字节流 ]──┬───[ ≥ 3.5 字符静默 ]────
                                   │  (字节间 ≤ 1.5)  │
                                   ▲                  ▲
                                帧起始 (Start)      帧结束 (End)
```

### 1.1 字符时间 $T_{\text{char}}$ 计算

在串口通信中，一个标准的 Modbus RTU 字符包含 **11 bits**：

$$
\text{Bits per Character} = \underbrace{1}_{\text{Start}} + \underbrace{8}_{\text{Data}} + \underbrace{1}_{\text{Parity/Stop}} + \underbrace{1}_{\text{Stop}} = 11\ \text{bits}
$$

> [!NOTE] 字符时长公式
> $$
> T_{\text{char}} = \frac{11}{\text{Baudrate}}\ \ (\text{秒})
> $$

### 1.2 临界波特率与固定时间法则（19200 bps 断点）

Modbus 官方规范特别规定：**当波特率 > 19200 bps 时，不再随波特率线性缩减时间，必须使用固定超时阈值**，以规避微控制器定时器分辨率不足及中断延迟抖动。

| 区域划分 | 波特率范围 | $t_{1.5}$ 字节超时阈值 | $t_{3.5}$ 帧定界静默阈值 |
| :------: | :--------: | :--------------------- | :----------------------- |
| **低速模式** | $\le 19200\ \text{bps}$ | 严格按 $1.5 \times T_{\text{char}}$ | 严格按 $3.5 \times T_{\text{char}}$ |
| **高速模式** | $> 19200\ \text{bps}$   | **固定 750 μs**                     | **固定 1.750 ms (1750 μs)**         |

#### 常用波特率理论时间速查表

| 波特率 (Baudrate) | 1 Bit 时长 $T_{\text{bit}}$ | 1 字符时长 $T_{\text{char}}$ | $t_{1.5}$ 超时 | $t_{3.5}$ 帧结束判定 |
| :--------------: | :-------------------------: | :--------------------------: | :------------: | :------------------: |
| **2400 bps**   | $416.7\ \mu s$ | $4.583\ \text{ms}$ | **6.875 ms**         | **16.042 ms**        |
| **4800 bps**   | $208.3\ \mu s$ | $2.292\ \text{ms}$ | **3.438 ms**         | **8.021 ms**         |
| **9600 bps**   | $104.2\ \mu s$ | $1.146\ \text{ms}$ | **1.719 ms**         | **4.010 ms**         |
| **19200 bps**  | $52.1\ \mu s$  | $0.573\ \text{ms}$ | **859 μs**           | **2.005 ms**         |
| **38400 bps**  | $26.0\ \mu s$  | $0.286\ \text{ms}$ | **750 μs** (固定)     | **1.750 ms** (固定)   |
| **115200 bps** | $8.68\ \mu s$  | $0.095\ \text{ms}$ | **750 μs** (固定)     | **1.750 ms** (固定)   |

> [!TIP] 工程取值建议
> - 定时器 ARR 建议按 **理论值 × 1.1 ~ 1.2** 取整，为中断抖动预留余量；
> - 高速模式下 750 μs / 1750 μs 属于 **强制阈值**，即便实际字符时间更短，也不能收紧；
> - 若使用 RTOS，任务调度周期必须 << $t_{1.5}$，否则轮询式 RX 会漏字节，建议直接走 ISR + DMA + 硬件 RTO。

---

## 📤 2. 发送（TX）控制与 DE 切换时序

### 2.1 标志位核心辨析：TXE vs TC

| 标志位 | 全称 | 触发时机 | 在 485 中的控制角色 |
| :----: | :--- | :------- | :------------------ |
| **`TXE`** | Transmit Data Register Empty | 发送数据寄存器空，数据已转入移位寄存器 | 仅用于通知 CPU/DMA 继续写入下一字节 |
| **`TC`**  | Transmission Complete        | **移位寄存器最后 1 bit（含停止位）彻底离开物理引脚** | **唯一合法拉低 DE/RE 切回接收的时机** |

> [!CAUTION] 致命误区：DMA 完成 ≠ 物理发送完成
> - **DMA TCIF 中断** 仅代表内存数据全部写入了串口数据寄存器（TDR），此时最后一个字节甚至尚未开始或刚刚开始移位发送；
> - 若在 DMA 中断中立即拉低 DE 引脚，**必将导致帧尾最后一个字节被物理截断**，从机收到残帧后 CRC 校验失败。

### 2.2 优雅的"交接棒"流程

```text
[启动发送]
   │
   ▼
[拉高 DE 引脚]  ── 切为 TX 模式
   │
   ▼
[启动 DMA 搬运数据]
   │
   ▼
[DMA 传输完成中断 (DMA TCIF)] ──► 维持 DE 为高，仅顺手开启串口 TCIE (TC 中断使能)
                                     │
                                     ▼
                              [串口 TC 中断触发] ──► 清除 TC 标志位，关闭 TCIE
                                                     立即拉低 DE 切回 RX 模式 ⬇
```

> [!TIP] 为什么 TX 发完后不需要加软件延时？
> 发送数据长度完全已知。通过 `TC` 标志位可以在最后一位发出的**微秒瞬间**立即释放总线。若在发完后加上软延时才释放 DE，高速响应的从机会立刻回包，导致主从机的驱动器在 RS-485 差分线上同时使能输出，造成 **总线竞争（Bus Contention）**，轻则波形畸变，重则烧毁收发器。

---

## 📥 3. 接收（RX）驱动状态机（FSM）

```text
                    收到首字节
        ┌──────────────────────────────────┐
        ▼                                  │
 ┌─────────────┐                      ┌─────────────┐
 │    IDLE     │                      │  RECEIVING  │
 │ (WAIT_FIRST)│                      │ (接收数据中) │
 └─────────────┘                      └─────────────┘
        ▲                                  │  │
        │                                  │  │ 字节间隔达到 3.5 字符
        │               字节间隔介于 1.5~3.5 字符 │
        │                                  │  ▼
        │                                  │ ┌────────────────┐
        │                                  │ │ FRAME_COMPLETE │
        │                                  │ │ (接收完成/待验) │
        │                                  │ └────────────────┘
        │                                  ▼        │
        │                            ┌─────────────┐│ CRC 校验失败
        │ 恢复完整 3.5 字符静默       │ERROR_SILENT ││ 或业务解析完毕
        └────────────────────────────│ (异常静默等待)│◄┘
                                     └─────────────┘
```

### 3.1 状态机核心处理规则

1. **`STATE_IDLE`（空闲等待）**
   - `DE = 0`（处于接收监听）；
   - 收到首字节后，重置并启动硬件超时定时器（TIM / RTO），进入 `STATE_RECEIVING`。

2. **`STATE_RECEIVING`（数据接收中）**
   - 每一个新字节到达，**立刻重置超时定时器清零重新计数**；
   - 若在定时器超过 $t_{1.5}$ 且小于 $t_{3.5}$ 之间突然有数据灌入：判定为**帧内断裂**，清空接收 Buffer，直接强转至 `STATE_ERROR_SILENT`；
   - 若总线持续静默，定时器达到 $t_{3.5}$ 阈值触发中断：说明整帧发送完成，进入 `STATE_FRAME_COMPLETE`。

3. **`STATE_FRAME_COMPLETE`（收帧完成）**
   - 关停超时定时器；
   - 读取 DMA 计数器计算真实收包长度：$\text{Length} = \text{BufferSize} - \text{CNDTR}$；
   - 提取报文并进行 **CRC-16 校验**。若校验失败，转入 `STATE_ERROR_SILENT`；通过则交由应用层解析，解析完毕切回 `STATE_IDLE`。

4. **`STATE_ERROR_SILENT`（异常静默同步）**
   - 发生校验错误或 $t_{1.5}$ 超时后，**绝不直接接收新数据**；
   - 丢弃所有新来的杂散字节，重新启动 $t_{3.5}$ 定时器，直到总线上完整出现一段没有任何信号的 $t_{3.5}$ 静默，才退回 `STATE_IDLE` 重新同步。

> [!WARNING] 常见踩坑
> - 收到 CRC 错误后直接 `state = IDLE`，会把下一帧的头几个字节误当作新帧起始，引发**连环误判**；
> - $t_{1.5}$ 与 $t_{3.5}$ 共用一个中断，未在 ISR 内区分"断裂"和"结束"两种情况；
> - 状态机切换未在临界区保护，主循环与 ISR 同时改写 `state` 引发**竞态**（表现为偶发漏帧、状态跳变）。

---

## 🛡️ 4. CRC-16 校验：算法、字节序与验证

### 4.1 校验范围与生成规则

Modbus RTU 使用 **CRC-16 / MODBUS** 变种，参数如下：

| 参数 | 值 | 备注 |
| :--- | :-: | :--- |
| 多项式 (Poly)   | `0x8005` ($x^{16}+x^{15}+x^{2}+1$) | 反转后为 `0xA001`，用于 LSB-first 查表 |
| 初始值 (Init)   | `0xFFFF` | **不是** 0x0000 |
| 输入反转 (RefIn)  | `true`  | 每个字节低位先进 |
| 输出反转 (RefOut) | `true`  | 结果整体低位在前 |
| 异或输出 (XorOut) | `0x0000` | 无最终异或 |
| 校验范围        | Slave ID → 数据段最后一个字节 | **不含**尾部 2 字节 CRC 自身 |

**核心性质**：将 **完整报文（含尾部 CRC）** 再算一遍 CRC-16，结果必为 `0x0000` —— 这是接收侧最优雅的校验方式。

$$
\text{CRC16}(\text{Frame}[0..N-1] \,\|\, \text{CRC}_{\text{LSB}} \,\|\, \text{CRC}_{\text{MSB}}) = 0
$$

### 4.2 字节序唯一例外规则

> [!WARNING] Modbus 协议唯一的逆序设计
> - **报文主体数据**（Slave ID、Function Code、寄存器地址、寄存器数值）：**大端序（Big-Endian / MSB First）**；
> - **CRC-16 校验码**：**小端序（Little-Endian / LSB First）** —— 与报文主体相反！

例如：对数据计算得到 CRC-16 值为 `0x4B37`：

| 报文位置 | 存放字节 | 说明 |
| :------: | :------: | :--- |
| `Buffer[Len - 2]` | `0x37` | CRC **低字节 LSB** 先发送 |
| `Buffer[Len - 1]` | `0x4B` | CRC **高字节 MSB** 后发送 |

### 4.3 位运算实现（无查表版，节省 Flash）

```c
/**
 * @brief Modbus CRC-16 位运算实现（无查表）
 * @param data   待校验数据指针
 * @param length 数据长度（字节）
 * @return       CRC-16 校验值（低字节先入报文）
 */
uint16_t Modbus_CRC16_Bitwise(const uint8_t *data, uint16_t length)
{
    uint16_t crc = 0xFFFF;
    for (uint16_t i = 0; i < length; i++)
    {
        crc ^= (uint16_t)data[i];
        for (uint8_t bit = 0; bit < 8; bit++)
        {
            if (crc & 0x0001)
            {
                crc = (crc >> 1) ^ 0xA001;
            }
            else
            {
                crc >>= 1;
            }
        }
    }
    return crc;
}
```

- **优势**：只需 512 B Flash，适合小容量 MCU（如 STM32F030C8）；
- **劣势**：每字节约 8 次循环，比查表法慢约 8 倍；115200 bps 下仍在预算内。

### 4.4 标准验证用例（用于自测 / 单元测试）

> [!EXAMPLE] 官方样例 —— 读取从机 0x01 的保持寄存器
> **请求帧（8 字节，含 CRC）**：
> ```text
> 01 03 00 00 00 01 84 0A
> └┬┘└┬┘└──┬──┘└──┬──┘└─┬─┘
>  │  │   │      │     └── CRC-16 (LSB=0x84, MSB=0x0A)
>  │  │   │      └──────── 寄存器数量 = 1
>  │  │   └─────────────── 起始寄存器地址 = 0x0000
>  │  └─────────────────── 功能码 = 0x03 (Read Holding Registers)
>  └────────────────────── 从机地址 = 0x01
> ```
> **计算过程**：对 `01 03 00 00 00 01` 六个字节做 CRC-16 → 得到 `0x0A84`
> **落盘顺序**：`Buffer[6] = 0x84` (LSB), `Buffer[7] = 0x0A` (MSB)

其它常用测试向量：

| 输入数据（Hex）         | CRC-16 结果 | 报文尾部（LSB, MSB） |
| :---------------------- | :---------: | :------------------: |
| `01 03 00 00 00 01`     | `0x0A84`    | `84 0A`              |
| `01 06 00 01 00 03`     | `0x5A9B`    | `9B 5A`              |
| `11 03 00 01 00 02`     | `0x909D`    | `9D 90`              |
| `01 10 00 01 00 02 04 00 0A 01 02` | `0xF5E9` | `E9 F5` |

> [!TIP] 快速自检
> 在单元测试里断言 `Modbus_CRC16(payload_with_crc, total_len) == 0x0000`，比分别比对 LSB/MSB 更不易出错。

### 4.5 校验失败常见根因

| 现象 | 根因 | 定位方法 |
| :--- | :--- | :------- |
| **CRC 完全随机** | DE 提前拉低导致帧尾字节被截断 | 用逻辑分析仪看 TX 波形最后一 bit 与 DE 下降沿的相对位置 |
| **CRC 高位固定错** | 发送时 CRC 用了大端序（MSB First） | 打印 `Buffer[Len-2]`、`Buffer[Len-1]`，与本文 4.4 表比对 |
| **偶发 CRC 错** | $t_{1.5}$ 阈值过紧，帧内断裂未被识别 | 提高阈值到 $1.5 \times T_{\text{char}} \times 1.2$，观察错误率变化 |
| **首字节被吃掉** | IDLE → RECEIVING 转换与 DMA 启动之间有空窗 | 使用 UART + DMA Circular 模式，或 STM32 硬件 RTO |
| **连续两帧都错** | CRC 失败后直接回 IDLE，把下一帧头当起始 | 严格按 §3 `ERROR_SILENT` 处理 |

---

## 💻 5. 完整 C 语言驱动代码实现（STM32 HAL 参考）

> [!NOTE] 代码风格
> 本节遵循项目 MCU 规范：**Allman 大括号**（`{` 独占一行）、4 空格缩进、`snake_case` 变量与 `PascalCase` 类型/函数。

### 5.1 快速查表法 CRC-16 实现

```c
#include <stdint.h>
#include <stdbool.h>

static const uint16_t crc16_table[256] = {
    0x0000, 0xC0C1, 0xC181, 0x0140, 0xC301, 0x03C0, 0x0280, 0xC241,
    0xC601, 0x06C0, 0x0780, 0xC741, 0x0500, 0xC5C1, 0xC481, 0x0440,
    0xCC01, 0x0CC0, 0x0D80, 0xCD41, 0x0F00, 0xCFC1, 0xCE81, 0x0E40,
    0x0A00, 0xCAC1, 0xCB81, 0x0B40, 0xC901, 0x09C0, 0x0880, 0xC841,
    0xD801, 0x18C0, 0x1980, 0xD941, 0x1B00, 0xDBC1, 0xDA81, 0x1A40,
    0x1E00, 0xDEC1, 0xDF81, 0x1F40, 0xDD01, 0x1DC0, 0x1C80, 0xDC41,
    0x1400, 0xD4C1, 0xD581, 0x1540, 0xD701, 0x17C0, 0x1680, 0xD641,
    0xD201, 0x12C0, 0x1380, 0xD341, 0x1100, 0xD1C1, 0xD081, 0x1040,
    0xF001, 0x30C0, 0x3180, 0xF141, 0x3300, 0xF3C1, 0xF281, 0x3240,
    0x3600, 0xF6C1, 0xF781, 0x3740, 0xF501, 0x35C0, 0x3480, 0xF441,
    0x3C00, 0xFCC1, 0xFD81, 0x3D40, 0xFF01, 0x3FC0, 0x3E80, 0xFE41,
    0xFA01, 0x3AC0, 0x3B80, 0xFB41, 0x3900, 0xF9C1, 0xF881, 0x3840,
    0x2800, 0xE8C1, 0xE981, 0x2940, 0xEB01, 0x2BC0, 0x2A80, 0xEA41,
    0xEE01, 0x2EC0, 0x2F80, 0xEF41, 0x2D00, 0xEDC1, 0xEC81, 0x2C40,
    0xE401, 0x24C0, 0x2580, 0xE541, 0x2700, 0xE7C1, 0xE681, 0x2640,
    0x2200, 0xE2C1, 0xE381, 0x2340, 0xE101, 0x21C0, 0x2080, 0xE041,
    0xA001, 0x60C0, 0x6180, 0xA141, 0x6300, 0xA3C1, 0xA281, 0x6240,
    0x6600, 0xA6C1, 0xA781, 0x6740, 0xA501, 0x65C0, 0x6480, 0xA441,
    0x6C00, 0xACC1, 0xAD81, 0x6D40, 0xAF01, 0x6FC0, 0x6E80, 0xAE41,
    0xAA01, 0x6AC0, 0x6B80, 0xAB41, 0x6900, 0xA9C1, 0xA881, 0x6840,
    0x7800, 0xB8C1, 0xB981, 0x7940, 0xBB01, 0x7BC0, 0x7A80, 0xBA41,
    0xBE01, 0x7EC0, 0x7F80, 0xBF41, 0x7D00, 0xBDC1, 0xBC81, 0x7C40,
    0xB401, 0x74C0, 0x7580, 0xB541, 0x7700, 0xB7C1, 0xB681, 0x7640,
    0x7200, 0xB2C1, 0xB381, 0x7340, 0xB101, 0x71C0, 0x7080, 0xB041,
    0x5000, 0x90C1, 0x9181, 0x5140, 0x9301, 0x53C0, 0x5280, 0x9241,
    0x9601, 0x56C0, 0x5780, 0x9741, 0x5500, 0x95C1, 0x9481, 0x5440,
    0x9C01, 0x5CC0, 0x5D80, 0x9D41, 0x5F00, 0x9FC1, 0x9E81, 0x5E40,
    0x5A00, 0x9AC1, 0x9B81, 0x5B40, 0x9901, 0x59C0, 0x5880, 0x9841,
    0x8801, 0x48C0, 0x4980, 0x8941, 0x4B00, 0x8BC1, 0x8A81, 0x4A40,
    0x4E00, 0x8EC1, 0x8F81, 0x4F40, 0x8D01, 0x4DC0, 0x4C80, 0x8C41,
    0x4400, 0x84C1, 0x8581, 0x4540, 0x8701, 0x47C0, 0x4680, 0x8641,
    0x8201, 0x42C0, 0x4380, 0x8341, 0x4100, 0x81C1, 0x8081, 0x4040
};

uint16_t Modbus_CRC16(const uint8_t *buffer, uint16_t length)
{
    uint16_t crc = 0xFFFF;
    while (length--)
    {
        uint8_t index = (uint8_t)(crc ^ *buffer++);
        crc = (crc >> 8) ^ crc16_table[index];
    }
    return crc;
}
```

### 5.2 状态机定义与硬件收发控制架构

```c
#include "stm32f4xx_hal.h"   /* 根据实际型号调整 */

#define RS485_RX_BUF_SIZE  256

typedef enum
{
    MODBUS_STATE_IDLE = 0,
    MODBUS_STATE_RECEIVING,
    MODBUS_STATE_FRAME_COMPLETE,
    MODBUS_STATE_ERROR_SILENT
} ModbusRxState_t;

typedef struct
{
    ModbusRxState_t state;
    uint8_t         rx_buf[RS485_RX_BUF_SIZE];
    uint16_t        rx_len;
} ModbusHandle_t;

extern UART_HandleTypeDef huart2;
extern TIM_HandleTypeDef  htim7;   /* 用于 t3.5 定时判定 */

ModbusHandle_t g_modbus;

#define RS485_TX_EN()  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_8, GPIO_PIN_SET)
#define RS485_RX_EN()  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_8, GPIO_PIN_RESET)

/* ================== 发送流程 (TX) ================== */
void Modbus_SendPacket(uint8_t *data, uint16_t len)
{
    uint16_t crc = Modbus_CRC16(data, len);

    /* Modbus CRC-16: 低字节在前 (LSB), 高字节在后 (MSB) */
    data[len]     = (uint8_t)(crc & 0xFF);
    data[len + 1] = (uint8_t)((crc >> 8) & 0xFF);
    len += 2;

    /* 1. 先切换 RS485 收发芯片引脚为 TX 模式 */
    RS485_TX_EN();

    /* 2. 清除 UART TC 标志位，确保状态干净 */
    __HAL_UART_CLEAR_FLAG(&huart2, UART_FLAG_TC);

    /* 3. 启动 DMA 发送 */
    HAL_UART_Transmit_DMA(&huart2, data, len);
}

/* DMA 发送完毕回调（此时硬件移位寄存器里还有最后一字节未走完引脚） */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART2)
    {
        /* 切忌在此处拉低 DE！仅开启串口发送完成中断 (TCIE) */
        __HAL_UART_ENABLE_IT(&huart2, UART_IT_TC);
    }
}
```

### 5.3 串口与定时器中断服务

```c
/* 串口中断服务函数 */
void USART2_IRQHandler(void)
{
    /* 判定是否是最后的物理停止位走完 */
    if (__HAL_UART_GET_FLAG(&huart2, UART_FLAG_TC) &&
        __HAL_UART_GET_IT_SOURCE(&huart2, UART_IT_TC))
    {
        __HAL_UART_DISABLE_IT(&huart2, UART_IT_TC);
        __HAL_UART_CLEAR_FLAG(&huart2, UART_FLAG_TC);

        /* 核心：所有 bit 已离开发送引脚，瞬间切回 RX 接收模式 */
        RS485_RX_EN();
        return;
    }

    /* 单字节接收处理（若使用 DMA + RTO 则由硬件外设自动搬运） */
    if (__HAL_UART_GET_FLAG(&huart2, UART_FLAG_RXNE))
    {
        uint8_t byte = (uint8_t)(huart2.Instance->DR & 0xFF);

        if (g_modbus.state == MODBUS_STATE_IDLE)
        {
            g_modbus.rx_len = 0;
            g_modbus.rx_buf[g_modbus.rx_len++] = byte;
            g_modbus.state = MODBUS_STATE_RECEIVING;

            /* 启动 t3.5 超时硬件定时器（例如 TIM7） */
            __HAL_TIM_SET_COUNTER(&htim7, 0);
            HAL_TIM_Base_Start_IT(&htim7);
        }
        else if (g_modbus.state == MODBUS_STATE_RECEIVING)
        {
            if (g_modbus.rx_len < RS485_RX_BUF_SIZE)
            {
                g_modbus.rx_buf[g_modbus.rx_len++] = byte;
            }
            /* 每收到一个字节，立即复位定时器计数值，重新开始倒计时 */
            __HAL_TIM_SET_COUNTER(&htim7, 0);
        }
        else if (g_modbus.state == MODBUS_STATE_ERROR_SILENT)
        {
            /* 出错静默期：有数据冲刷进来，直接丢弃并重新等待 3.5 字符静默 */
            __HAL_TIM_SET_COUNTER(&htim7, 0);
        }
    }
}

/* 硬件定时器 (TIM7) 达到 t3.5 阈值溢出中断 */
void TIM7_IRQHandler(void)
{
    if (__HAL_TIM_GET_FLAG(&htim7, TIM_FLAG_UPDATE))
    {
        __HAL_TIM_CLEAR_IT(&htim7, TIM_IT_UPDATE);
        HAL_TIM_Base_Stop_IT(&htim7);   /* 停止计时 */

        if (g_modbus.state == MODBUS_STATE_RECEIVING)
        {
            /* 总线已维持 t3.5 静默，一帧数据接收完成 */
            g_modbus.state = MODBUS_STATE_FRAME_COMPLETE;
        }
        else if (g_modbus.state == MODBUS_STATE_ERROR_SILENT)
        {
            /* 异常后已重新获得完整的 3.5 字符静默，总线重新同步成功 */
            g_modbus.state = MODBUS_STATE_IDLE;
        }
    }
}
```

### 5.4 主轮询 / 任务处理

```c
void Modbus_Process_Loop(void)
{
    if (g_modbus.state != MODBUS_STATE_FRAME_COMPLETE)
    {
        return;
    }

    /* 最短 Modbus 报文：地址(1) + 功能码(1) + CRC(2) */
    if (g_modbus.rx_len >= 4)
    {
        uint16_t rx_crc = (uint16_t)g_modbus.rx_buf[g_modbus.rx_len - 2] |
                          ((uint16_t)g_modbus.rx_buf[g_modbus.rx_len - 1] << 8);

        /* 排除尾部 2 字节 CRC，计算纯数据段的校验值 */
        if (Modbus_CRC16(g_modbus.rx_buf, g_modbus.rx_len - 2) == rx_crc)
        {
            /* 校验通过，派发到业务逻辑解析 */
            /* Modbus_Parse_Execute(g_modbus.rx_buf, g_modbus.rx_len); */
            g_modbus.state = MODBUS_STATE_IDLE;
            return;
        }
    }

    /* CRC 校验失败或长度不足，进入错误静默状态重新同步 */
    g_modbus.state = MODBUS_STATE_ERROR_SILENT;
    __HAL_TIM_SET_COUNTER(&htim7, 0);
    HAL_TIM_Base_Start_IT(&htim7);
}
```

---

## ✅ 6. 落地校验清单（Checklist）

> [!SUCCESS] 上线前必过
> - [ ] **TX 时序**：`DE ↑` → DMA Start → DMA TCIF → `TCIE enable` → TC ISR → `DE ↓`，用示波器确认最后停止位与 DE 下降沿间隔 < 1 μs
> - [ ] **RX 定时器**：$t_{1.5}$ / $t_{3.5}$ 阈值按波特率分区取值（≤19200 计算，>19200 固定 750 μs / 1750 μs）
> - [ ] **CRC 范围**：Slave ID → 数据段末尾，**不含** CRC 自身
> - [ ] **CRC 字节序**：**低字节在前**，高字节在后（唯一逆序字段）
> - [ ] **CRC 单测**：`Modbus_CRC16(frame_with_crc, total_len) == 0x0000`
> - [ ] **错误恢复**：CRC 失败后进入 `ERROR_SILENT`，等待完整 $t_{3.5}$ 静默再复位
> - [ ] **并发保护**：`state` / `rx_len` 主循环与 ISR 共访问，做临界区（关中断 / atomic）
> - [ ] **总线偏置**：开路时 A/B 差分电压 ≥ 200 mV（配合 470 Ω ~ 1 kΩ 上下拉电阻）
> - [ ] **终端匹配**：总线两端各 120 Ω，中间节点严禁并联终端电阻

---

## 🧠 思考与延伸索引

- [[Modbus_Exception_Codes]] — 常见异常码（`01` 非法功能、`02` 非法数据地址、`03` 非法数据值）的定义与标准应答格式
- [[RS485_Hardware_Biasing]] — 为什么总线开路时必须配置 $470\ \Omega \sim 1\ \text{k}\Omega$ 上拉与下拉电阻，以防止总线悬浮引发的误中断
- [[STM32_USART_RTO_Native]] — 利用 STM32 高级 USART 外设的 Hardware Receiver Timeout 功能彻底解放通用定时器
- [[Modbus_ASCII_vs_RTU]] — ASCII 模式使用 LRC 校验与 CRLF 定界，与 RTU 完全互不兼容
- [[Modbus_TCP_Frame]] — TCP 模式去掉 CRC，改用 MBAP Header 中的 Length 字段定界

---

## 📚 参考资料

1. **Modbus Organization** — *Modbus over Serial Line Specification and Implementation Guide V1.02*, 2006.
2. **STMicroelectronics** — *RM0090 Reference Manual: STM32F4 Series*（USART / DMA / TIMER 章节）.
3. **STMicroelectronics** — *AN4839: Level translation techniques for serial interfaces*.
4. **TI Application Report SLLA272** — *The RS-422 and RS-485 Standards Guide and Their Applications*.
