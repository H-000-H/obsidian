---
title: Modbus 协议软件模拟（Python）完整实现手册
tags:
  - Embedded
  - Protocol
  - Modbus
  - ModbusRTU
  - ModbusTCP
  - Python
  - Simulation
  - CRC16
status: Completed
---

# 🐍 Modbus 协议软件模拟（Python）完整实现手册

> [!INFO] 笔记概览
> - **目的**：用 **Python 软件模拟**把 RTU / ASCII / TCP 帧、CRC-16/MODBUS、LRC、异常应答在逻辑层建出来，理解"主→从一问一答"时序与帧格式，便于仿真调试。
> - **内容范围**：CRC/LRC 校验 · RTU 帧编解码 · 功能码打包 · 异常应答 · ASCII 模式 · TCP MBAP · 主从一问一答 · 超时重试
> - **推荐路径**：§1 校验 → §2 RTU 帧 → §3 功能码 → §4 异常 → §5 ASCII → §6 TCP → §7 主从示例 → §8 时序
> - **免责声明**：真实对接设备建议直接用宿主库（串口 `pyserial`、Modbus `pymodbus`），本文件用于理解协议本身。

---

## 🗺️ 0. 整体关系

```text
主站请求(功能码+数据)
    ↓
帧编码器：加地址 + 校验(CRC/LRC) → RTU 或 ASCII 帧
    ↓
串口/网络发送
    ↓
从站拆帧 → 校验 → 组应答（或异常） → 发回
    ↓
主站校验应答 / 超时重试
```

---

## 🛡️ 1. CRC-16/MODBUS 与 LRC

RTU 用 **CRC-16/MODBUS**：多项式 `0x8005` 反映射即 `0xA001`，初值 `0xFFFF`，**先发低字节再发高字节**。

```python
def crc16_modbus(data):
    """对字节序列计算 CRC-16/MODBUS，返回完整 16 位值。"""
    crc = 0xFFFF
    for b in data:
        crc ^= b
        for _ in range(8):
            crc = ((crc >> 1) ^ 0xA001) if (crc & 1) else (crc >> 1)
    return crc

def crc_bytes(data):
    """返回 RTU 帧末尾要追加的 2 个字节：先低后高。"""
    c = crc16_modbus(data)
    return [(c & 0xFF), (c >> 8) & 0xFF]
```

LRC 用于 **ASCII** 模式，是各字节和的补码，只取低 8 位：

```python
def lrc(data):
    total = sum(data) & 0xFF
    return (-total) & 0xFF          # 补码：对低 8 位取反加一
```

> [!EXAMPLE] 校验自检
> ```python
> assert crc16_modbus([0x01, 0x03, 0x00, 0x00, 0x00, 0x02]) == 0x0BC4  # 帧尾 C4 0B
> assert lrc([0x01, 0x03, 0x00, 0x00, 0x00, 0x02]) == 0xFA
> ```
> CRC 完整算法剖析与更多测试向量见 [[modbus]] §4、协议层字节序规则见 [[Modbus]] §6。

---

## 📦 2. RTU 帧：构造与解析

RTU 帧 = `地址 | 功能码 | 数据 | CRC低 | CRC高`，地址 0 是广播。

```python
def build_rtu(slave: int, func: int, data=b""):
    """组一帧 RTU。data 是已拼好的请求/应答数据区。"""
    body = bytes([slave & 0xFF, func]) + bytes(data)
    return body + bytes(crc_bytes(body))

def parse_rtu(frame: bytes):
    """解析 RTU 帧；地址不符或 CRC 错返回 None。"""
    if len(frame) < 4:
        return None
    slave, func = frame[0], frame[1]
    payload = frame[2:-2]
    if crc16_modbus(frame[:-2]) != (frame[-2] | (frame[-1] << 8)):
        return None
    return {"slave": slave, "func": func, "data": payload}
```

> [!NOTE] 数据区字节序
> 数据区里 16 位数量、地址都用**大端**（高字节在前），与 CRC 的小端相反：
> ```python
> def u16(v):
>     return [(v >> 8) & 0xFF, v & 0xFF]
> ```

---

## 🔢 3. 常用功能码数据打包

### 3.1 读保持寄存器（0x03）

请求 = `起始寄存器(2B) + 数量(2B)`；应答 = `字节数(1B) + 寄存器值(2B × n)`（寄存器本身大端）。

```python
def read_holding_request(slave, start, count):
    return build_rtu(slave, 0x03, u16(start) + u16(count))

def read_holding_response(slave, values):
    """values: 一组 16 位寄存器值。"""
    data = [len(values) * 2]
    for v in values:
        data += u16(v)
    return build_rtu(slave, 0x03, data)
```

### 3.2 写单寄存器（0x06）

请求 = `寄存器地址(2B) + 值(2B)`；应答**原样回显**请求。

```python
def write_single_register(slave, reg, value):
    return build_rtu(slave, 0x06, u16(reg) + u16(value))
```

### 3.3 读线圈（0x01）

应答把连续 bit 按位打包成字节，最后一个字节高位补 0。

```python
def read_coil_request(slave, start, count):
    return build_rtu(slave, 0x01, u16(start) + u16(count))

def coil_pack(bits):
    """把 bool 列表打包成字节序列（LSB 在前）。"""
    out = []
    for i in range(0, len(bits), 8):
        byte = 0
        for j, b in enumerate(bits[i:i+8]):
            if b:
                byte |= (1 << j)
        out.append(byte)
    return out

def read_coil_response(slave, bits):
    packed = coil_pack(bits)
    return build_rtu(slave, 0x01, bytes([len(packed)]) + bytes(packed))
```

---

## ⚠️ 4. 异常应答

从站无法完成时，回功能码 **+0x80** 再加 1 字节异常码。

```python
EX_BAD_FUNC   = 0x01   # 非法功能码
EX_BAD_ADDR   = 0x02   # 非法数据地址
EX_BAD_VALUE  = 0x03   # 非法数据值
EX_DEV_FAIL   = 0x04   # 从站设备故障

def exception_response(slave, func, code):
    return build_rtu(slave, (func | 0x80), bytes([code]))
```

> [!CAUTION] 广播不回包
> 广播（地址 0）请求从站一律不回包，也不回异常。异常码完整定义见 [[Modbus]] §7。

---

## 🔤 5. ASCII 模式

ASCII 帧：以 `':'` 起，把每字节转成 2 个十六进制字符，末尾附 LRC 字符并 `CR LF` 结束。

```python
HEX = "0123456789ABCDEF"

def ascii_encode(buf: bytes) -> bytes:
    out = bytearray([ord(':')])
    for b in buf:
        out += HEX[b >> 4].encode() + HEX[b & 0x0F].encode()
    body = bytes(buf)
    out += HEX[lrc(body) >> 4].encode() + HEX[lrc(body) & 0x0F].encode()
    out += b'\r\n'
    return bytes(out)

def ascii_decode(frame: bytes):
    """把 : 起、CRLF 止的 ASCII 帧还原为二进制，并校验 LRC。"""
    txt = frame[1:-2]                       # 去掉 ':' 和 CRLF
    raw = bytes.fromhex(txt.decode())       # 十六进制字符 -> 二进制
    lrc_raw = raw[-1]
    if lrc(raw[:-1]) != lrc_raw:
        return None
    return raw[:-1]                          # 去掉末尾 LRC 字节
```

> [!EXAMPLE] ASCII 编码示例
> 读保持寄存器请求 `01 03 00 00 00 02`，LRC=`FA`：
> ```python
> assert ascii_encode(bytes([0x01, 0x03, 0x00, 0x00, 0x00, 0x02])) == \
>        b":010300000002FA\r\n"
> ```

---

## 🌐 6. Modbus TCP（MBAP）

TCP 帧 = MBAP 头(7B) + PDU（**无地址、无 CRC**），MBAP = `事务ID(2) + 协议ID(2,=0) + 长度(2) + 单元ID(1)`。长度 = PDU 字节数 + 1（加单元 ID）。

```python
import struct

def mbap_encode(unit, func, data=b""):
    pdu = bytes([func]) + bytes(data)
    head = struct.pack(">HHHB", 0x0001, 0x0000, len(pdu) + 1, unit)
    return head + pdu

def mbap_decode(frame: bytes):
    tid, pid, length, unit = struct.unpack(">HHHB", frame[:7])
    assert pid == 0
    pdu = frame[7:7 + length - 1]
    return {"tid": tid, "unit": unit, "func": pdu[0], "data": pdu[1:]}
```

> [!NOTE] TCP 为何无 CRC
> 可靠传输和差错由 TCP + MBAP 的 Length 字段负责，因此去掉了 RTU 的 CRC。帧结构对照见 [[Modbus]] §3.3。

---

## 🔄 7. 主站 → 从站 一问一答示例

模拟一个只有 4 个保持寄存器的从站，主站先读后写：

```python
class Slave:
    """内存型 Modbus 从站，只实现保持寄存器读写。"""
    def __init__(self, address=1, registers=None):
        self.address = address
        self.registers = list(registers or [0, 0, 0, 0])

    def handle(self, frame: bytes):
        """收到一帧 RTU，返回应答帧；广播或地址不对返回 None(不回)。"""
        msg = parse_rtu(frame)
        if msg is None:
            return None                     # CRC/格式错误：静默丢弃
        if msg["slave"] != self.address:
            if msg["slave"] == 0:           # 广播：处理但不回包
                self._apply_broadcast(msg)
            return None
        try:
            return self._reply(msg)
        except KeyError:
            return exception_response(self.address, msg["func"], EX_BAD_ADDR)

    def _apply_broadcast(self, msg):
        pass                                # 例如写多寄存器时改数据但不回

    def _reply(self, msg):
        d = msg["data"]
        func = msg["func"]
        if func == 0x03:                    # 读保持寄存器
            start = (d[0] << 8) | d[1]
            count = (d[2] << 8) | d[3]
            vals = self.registers[start:start + count]
            if len(vals) != count:
                raise KeyError
            return read_holding_response(self.address, vals)
        if func == 0x06:                    # 写单寄存器
            reg = (d[0] << 8) | d[1]
            val = (d[2] << 8) | d[3]
            self.registers[reg] = val
            return write_single_register(self.address, reg, val)
        raise KeyError


def run_master(slave, requests):
    """模拟一问一答：对每个请求打印请求与应答字节。"""
    for desc, frame in requests:
        print(desc, frame.hex().upper())
        resp = slave.handle(frame)
        print("   (从站不回包)" if resp is None else "   -> " + resp.hex().upper())


slave = Slave(1, [0x1234, 0xABCD, 0, 0])
requests = [
    ("读保持寄存器(0x03)", read_holding_request(1, 0, 2)),
    ("写单寄存器(0x06)写 0x0102 到 reg2", write_single_register(1, 2, 0x0102)),
    ("再读 4 个寄存器", read_holding_request(1, 0, 4)),
    ("读超出范围 -> 异常 02", read_holding_request(1, 4, 1)),
]
run_master(slave, requests)
```

输出示意（CRC 由 `crc_bytes` 实际算出，此处部分为示意值）：

```text
读保持寄存器(0x03)      010300000002C40B
   -> 0103041234ABCD0020
写单寄存器(0x06)        010602000102C9F8
   -> 010602000102C9F8          （原样回显）
再读 4 个寄存器         010300000004444D
   -> 0103081234ABCD01020000xxxx
读超出范围 -> 异常 02   010300040001xxxx
   -> 018302C0F1                 （功能 0x83 + 异常码 02）
```

---

## ⏱️ 8. 时序与重试

主站需自己管"多少时间没收到应答就重试"。把 RTU 帧间隔换算成**字符时间**：

```python
def char_us(baud):
    """一个字符(10bit: 8数据+1起始+1停止)的时长(微秒)。"""
    return 1_000_000 * 10 / baud

def frame_gap_us(baud):
    """RTU 两帧之间要求静默 ≥ 3.5 个字符时间。"""
    return 3.5 * char_us(baud)

print(frame_gap_us(9600))    # ≈ 3645 µs
```

> [!TIP] 最本质的时序
> **主站发出请求后，在超时窗口内若收到且 CRC 通过即成功；否则重试几次后报错。** 这与底层 UART 位时序叠加即可还原完整链路。临界波特率（19200 断点）与固定 750 µs / 1750 µs 阈值见 [[modbus]] §1。

---

## 🧭 9. 一图理解整体

```text
主站                              从站
 │  广播(地址0)：不回包             │
 │ ──────────────────────────────→ │
 │  正常请求: 地址+功能+数据+CRC     │
 │ ──────────────────────────────→ │  校验CRC → 执行操作
 │  应答:    地址+功能+数据+CRC     │
 │ ←────────────────────────────── │
 │  异常:    地址+功能|80+异常码    │
 │ ←────────────────────────────── │
 │  超时 → 重试(通常3次)            │
```

- RTU 两帧之间静默 ≥ 3.5 字符时间；ASCII 帧间静默 ≥ 1 秒。
- 从站若收到功能码不认识、地址越界、数据非法，回**异常帧**而非静默。

---

## 🔗 思考与延伸索引

- [[Modbus]] — Modbus 协议层帧结构、功能码、数据模型总览
- [[modbus]] — RS-485 Modbus RTU 硬件驱动与时序（C 语言实现）
- [[UART-code]] — Modbus RTU 承载的底层 UART 收发模拟
- [[CAN-code]] — 另一种总线的软件模拟实现对照
- [[DMA、Buffer 与流量控制]] — RX 帧定界与流控在真实驱动中的落地

---

## 📚 参考资料

1. **Modbus Organization** — *MODBUS Application Protocol Specification V1.1b3*, 2012.
2. **Modbus Organization** — *Modbus over Serial Line Specification V1.02*, 2006.
3. **Python** — `struct`、`pyserial`、`pymodbus` 官方文档.
4. 项目内部笔记：[[Modbus]]、[[modbus]]。
