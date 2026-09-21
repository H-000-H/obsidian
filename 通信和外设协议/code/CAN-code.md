---
title: CAN / CAN-FD 协议软件模拟（Python）完整实现手册
tags:
  - Embedded
  - Protocol
  - CAN
  - CANFD
  - Python
  - Simulation
  - BitStuffing
  - Arbitration
status: Completed
created: 2026-09-20
---

# 🐍 CAN / CAN-FD 协议软件模拟（Python）完整实现手册

> [!INFO] 笔记概览
> - **目的**：只提供 **Python 软件模拟**，不写 C、不碰晦涩的 CAN 寄存器。CAN 控制器寄存器各家差异极大、时序细节隐蔽，用软件把"总线逻辑 + 位填充 + CRC + 仲裁 + 双速率"在逻辑层建出来，比看寄存器更直观，也方便仿真调试。
> - **内容范围**：逻辑位↔物理电平 · CRC-15/17/21 · 位填充 · 数据帧编码 · 总线仲裁模型 · 解码校验 · CAN-FD 编码 · 双速率时序
> - **推荐路径**：§1 电平 → §2 CRC → §3 位填充 → §4 编码 → §5 仲裁 → §6 解码 → §7 CAN-FD → §8 双速率
> - **免责声明**：真实硬件收发建议直接用 `python-can` + `cantools`（DBC 解析）对接 CAN / CAN-FD 接口。本文件用于理解协议本身。

---

## 🗺️ 0. 整体关系

```text
数据/配置 (arb, data, BRS, bit rate)
    ↓
帧编码器（普通 CAN 或 CAN-FD）
    ↓
位填充(Stuffing) + CRC
    ↓
逻辑位序列 [0/1]
    ↓
物理电平模型（dominant/recessive → CAN_H/CAN_L）
    ↓
总线仲裁/时序模拟 / 真实总线
```

---

## ⚡ 1. 逻辑位与物理电平

CAN 只有两种电平：**显性 Dominant = 逻辑 0**，**隐性 Recessive = 逻辑 1**。显性覆盖隐性（线与）。

```python
def logical_to_level(bit: int):
    """把逻辑位映射成物理电平。
    返回: (CAN_H, CAN_L) 两条线的理想电压(V)。
    显性 0 -> CAN_H 高、CAN_L 低；隐性 1 -> 两条线都 2.5V。
    """
    if bit == 0:   # dominant
        return (3.5, 1.5)
    return (2.5, 2.5)   # recessive

def diff_voltage(bit: int) -> float:
    """差分电压：显性约 2V，隐性约 0V。"""
    return 2.0 if bit == 0 else 0.0
```

> [!CAUTION] 易错点
> CAN 的"逻辑 0"是显性、优先级高，和"0 表示空/低"的直觉相反。详见 [[CAN]] §1。

---

## 🛡️ 2. CRC：普通 CAN 与 CAN-FD

CRC 都在位级上做，输入是**报文从 SOF 到数据段末尾**这段逻辑位（不含后面的分隔符）。通用 LFSR 实现（MSB 先行、检查最高位再异或多项式）：

```python
def crc_div(bits, poly: int, deg: int) -> int:
    mask = (1 << deg) - 1
    crc = 0
    for b in bits:
        msb = (crc >> (deg - 1)) & 1   # 移位前寄存器最高位（被移出的一位）
        crc = ((crc << 1) & mask) | (b & 1)
        if msb:
            crc ^= poly
            crc &= mask
    return crc
```

三种 CRC 参数（按 ISO 11898 常见取值）：

```python
def crc15_can(bits):            # 普通 CAN：poly=0x4599, 无 init, xorout=0x7FFF
    return crc_div(bits, 0x4599, 15) ^ 0x7FFF

def crc17_fd(bits):             # CAN-FD（长度<=16字节）：poly=0x3850F
    return crc_div(bits, 0x3850F, 17)

def crc21_fd(bits):             # CAN-FD（长度>16字节）：poly=0x1DEDEE
    return crc_div(bits, 0x1DEDEE, 21)
```

把整数 CRC 展开成位列表（MSB 先传）：

```python
def to_bits(value: int, n: int):
    return [(value >> i) & 1 for i in range(n - 1, -1, -1)]
```

> [!WARNING] CRC 参数需交叉验证
> 上述 CRC 参数按 ISO 11898 常见取值给出，供理解和自测；真实接入前请以目标控制器/参考链路计算结果为准做一次交叉验证。

---

## 🔁 3. 位填充 Stuffing

普通 CAN：从 SOF 到 CRC 段每产生连续 5 个相同位，就插 1 个反向位打断，保证总线有足够跳变沿用于重同步。

```python
def stamp(bits):
    """插入填充位：发送每连续 5 个相同位后追加 1 个反向位。"""
    out = []
    streak = 0
    last = bits[0]
    for b in bits:
        if b == last:
            streak += 1
        else:
            streak = 1
            last = b
        out.append(b)
        if streak == 5:              # 已连续 5 个相同
            out.append(1 - b)        # 插反向位打断
            streak = 0
            last = 1 - b             # 填充位自身极性成为新起点
    return out

def unstuff(bits):
    """去填充：连续 5 个相同位后的第 6 位是填充位，跳掉。"""
    out = []
    streak = 0
    last = bits[0]
    i = 0
    n = len(bits)
    while i < n:
        b = bits[i]
        if i and b == last:
            streak += 1
        else:
            streak = 1
            last = b
        i += 1
        out.append(b)
        if streak == 5:              # 下一位是填充位，跳过
            if i < n:
                i += 1
            streak = 0
            last = b
    return out
```

> [!EXAMPLE] 往返自检
> ```python
> raw_test = [0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 0]
> assert unstuff(stamp(raw_test)) == raw_test
> print("stuff/unstuff round-trip OK,",
>       "填后", len(stamp(raw_test)), "位, 原", len(raw_test), "位")
> ```

---

## 📤 4. 普通 CAN：数据帧编码

标准帧（11 位 ID）数据帧字段：

```text
SOF | Identifier(11) | RTR | IDE | r0 | DLC(4) | Data(0~8B) | CRC(15) | CRC分隔 | ACK槽 | ACK分隔 | EOF(7) | IFS(3)
```

编码流程：先拼"参与 CRC 的原始位" → 算 CRC → 整体位填充 → 补后续固定字段。

```python
import itertools


def classic_data_frame(arb: int, data: bytes):
    """标准帧 11 位 ID + RTR=0(数据帧) + IDE=0 + r0=0。"""
    if not (0x000 <= arb <= 0x7FF):
        raise ValueError("11-bit ID 范围 0x000~0x7FF")
    if len(data) > 8:
        raise ValueError("普通 CAN 最多 8 字节")

    idb  = to_bits(arb, 11)
    dlcb = to_bits(len(data), 4)
    datab = list(itertools.chain.from_iterable(to_bits(b, 8) for b in data))

    # SOF + ID + RTR(0) + IDE(0) + r0(0) + DLC + 数据 —— 这整段参与 CRC
    raw = [0] + idb + [0, 0, 0] + dlcb + datab
    crc = crc15_can(raw)
    crcb = to_bits(crc, 15)

    # 位填充：SOF..CRC 段（含 CRC 位）
    stuffed = stamp(raw + crcb)

    # 之后是固定字段（不填充）
    return (stuffed   # 填充段
            + [1]        # CRC 分隔
            + [0]        # ACK 槽（接收节点在此填写，模拟成已 ACK=显性）
            + [1]        # ACK 分隔
            + [1] * 7    # EOF
            + [1] * 3)   # IFS
```

> [!TIP] 观察填充效果
> ```python
> frame = classic_data_frame(0x100, b"\x01\x02")
> print("填充后帧长(bit):", len(frame))
> print("".join(map(str, frame[:24])), "...")
> ```
> 帧长随填充位数变化，以实际运行为准。字段完整定义见 [[CAN]] §4。

---

## ⚖️ 5. 普通 CAN：总线仲裁模型

总线是多主"线与"，任意节点发显性(0)即总线显性：

```python
def bus_level(offered):
    """offered: 各节点要发送的位组成的列表；0 显性优先。"""
    return 0 if 0 in offered else 1

def arbitrate(frames):
    """frames: [[节点名, 位列表, 是否已退出]]；返回(赢家名字, 最终帧位序列)。
    节点边发边收，发隐性(1)却读到显性(0) → 失败退出。
    """
    bits = len(frames[0][1])
    result = []
    for i in range(bits):
        fade = [f for f in frames if not f[2]]        # 仍未退出的活节点
        won_bit = bus_level([f[1][i] for f in fade])  # 总线被拉成什么电平
        result.append(won_bit)
        for f in fade:
            if f[1][i] == 1 and won_bit == 0:         # 我发隐性，总线却显性
                f[2] = True                           # 仲裁失败退出
    winner = [name for name, _, lost in frames if not lost]
    return winner[0], result
```

> [!EXAMPLE] 两节点同发，ID 小者优先
> ```python
> n_a = ("A(id=0x100)", classic_data_frame(0x100, b"\x01"))
> n_b = ("B(id=0x0FF)", classic_data_frame(0x0FF, b"\x02"))
> frames = [[name, f, False] for name, f in (n_a, n_b)]
> winner, bus_seq = arbitrate(frames)
> print("赢得总线:", winner)
> print("总线前 16 位:", "".join(map(str, bus_seq[:16])))
> ```
> ID 越小 → 高位显性越早出现 → 优先级越高。`0x0FF < 0x100`，B 胜出。仲裁原理见 [[CAN]] §3。

---

## 📥 6. 接收 / 解码与校验

解码是编码逆过程：去填充 → 校验 CRC → 拆字段。按标准帧固定字段长度解析：

```python
# 发送端顺序（编码时已确定）:
# SOF(1) ID(11) RTR(1) IDE(1) r0(1) DLC(4) Data(DLC*8)
# CRC(15) CRC分隔(1) ACK槽(1) ACK分隔(1) EOF(7) IFS(3)
ID_N = 11
BEFORE_DLC = 1 + ID_N + 3        # SOF + ID + RTR + IDE + r0
DLC_AT = BEFORE_DLC              # DLC 从该下标开始
CRC_N = 15
CRC_DELIM = 1
ACK_SLOT = 1
ACK_DELIM = 1
EOF = 7
IFS = 3
TAIL = CRC_DELIM + ACK_SLOT + ACK_DELIM + EOF + IFS   # 不参与 CRC/去填充的尾部


def to_int(bits):
    v = 0
    for b in bits:
        v = (v << 1) | b
    return v
```

```python
from collections import namedtuple

Parsed = namedtuple("Parsed", ["id", "dlc", "data", "crc_ok"])


def parse_classic(frame: list):
    """解析普通 CAN 标准帧（已含全部填充与尾部）。"""
    # 1) 尾部是固定字段，先剥掉
    stuffed_len = len(frame) - TAIL
    stuffed_body = frame[:stuffed_len]

    # 2) 去填充 → 得到原始位（含 CRC 位）
    raw = unstuff(stuffed_body)

    # 3) 数据长度由 DLC 确定
    dlc = to_int(raw[DLC_AT:DLC_AT + 4])
    data_start = DLC_AT + 4
    data_end = data_start + dlc * 8
    crc_start = data_end

    id_val = to_int(raw[1:1 + ID_N])          # 跳过 SOF

    # 4) CRC 校验：对 "SOF..数据" 段本地算 CRC，与收到的 15 位 CRC 比
    calc = crc15_can(raw[:data_end])
    recv_crc = to_int(raw[crc_start:crc_start + CRC_N])
    crc_ok = calc == recv_crc

    datab = raw[data_start:data_end]
    data = bytes(to_int(datab[j:j + 8]) for j in range(0, len(datab), 8))

    return Parsed(id_val, dlc, data, crc_ok)


def parse_demo():
    orig = classic_data_frame(0x100, b"\x01\x02")
    p = parse_classic(orig)
    print("ID=0x%03X dlc=%d data=%s crc_ok=%s" %
          (p.id, p.dlc, list(p.data), p.crc_ok))


parse_demo()   # 期望: ID=0x100 dlc=2 data=[1, 2] crc_ok=True
```

> [!NOTE] CRC 不符的排查
> 去填充后 `raw[:data_end]` 正是算 CRC 的字段；与收到的 15 位 CRC 比对即可。若 CRC 不符，多半是位填充边界对不上（真实控制器会对填充错误直接报错误帧）。

---

## 🚀 7. CAN-FD：编码（FDF / BRS / ESI / DLC 映射）

CAN-FD 相比经典多了：

| 特性 | 说明 |
| :--- | :--- |
| **FDF=1（隐性）** | 标记 FD 帧，替换经典 r0 |
| **res** | 保留位（FD 为隐性 1） |
| **BRS** | 1 → 数据段提速 |
| **ESI** | 发送方错误被动时显性，否则隐性 |
| **DLC 扩展映射** | 9~15 → 12/16/20/24/32/48/64 字节 |

```python
FD_DLC_MAP = {
    9: 12, 10: 16, 11: 20, 12: 24,
    13: 32, 14: 48, 15: 64,
}


def fd_dlc_for_length(n: int) -> int:
    if n <= 8:
        return n
    for dlc, size in FD_DLC_MAP.items():
        if n <= size:
            return dlc
    raise ValueError("CAN-FD 最大 64 字节")


def canfd_data_frame(arb: int, data: bytes, brs: bool = True, esi: bool = False):
    """CAN-FD 帧思路（仅 11 位 ID 主段示例，不展开完整 29 位）。
    返回三段 (control, data_phase, tail)，便于按 nominal/data 两种位速率分别计时。
    """
    if len(data) > 64:
        raise ValueError("CAN-FD 最多 64 字节")
    dlc = fd_dlc_for_length(len(data))
    datab = list(itertools.chain.from_iterable(to_bits(b, 8) for b in data))

    # 仲裁/控制段参与 CRC 的原始位:
    #  SOF + ID(11) + FDF(1) + res(1) + BRS(1) + ESI(1) + DLC(4)
    raw = [0] + to_bits(arb, 11) + [1, 1] + [1 if brs else 0, 1 if esi else 0] + to_bits(dlc, 4)

    # 数据段：数据 + CRC。CRC 按长度选 17/21
    crc_field_len = 17 if len(data) <= 16 else 21
    raw_for_crc = raw + datab
    crc = crc17_fd(raw_for_crc) if crc_field_len == 17 else crc21_fd(raw_for_crc)
    crcb = to_bits(crc, crc_field_len)

    control = raw + datab                # 还在仲裁速率
    data_phase = crcb                    # CAN-FD 数据段（数据+CRC 在数据速率）
    tail = [1] + [0] + [1] + [1] * 7 + [1] * 3   # CRC分隔 + ACK槽 + ACK分隔 + EOF + IFS

    return control, data_phase, tail
```

> [!NOTE] 分段返回的用意
> `canfd_data_frame` 返回三段，便于 §8 按 nominal / data 两种位速率分别计时间。DLC 映射表与 CAN-FD 完整字段见 [[CAN]] §6、§8。

---

## ⏱️ 8. CAN-FD：双速率时序模拟

给每个 bit 按所处相位决定它的时长：

```python
def fd_timeline(control, data_phase, tail,
                t_nominal_us=2.0,     # 500 kbit/s -> 2.0 us/bit
                t_data_us=0.2,        # 5 Mbit/s   -> 0.2 us/bit
                sample_nominal=0.75): # 采样点位置（比例）
    events = []          # (起始, 结束, 相位, 电平)
    phases = [("nominal", 0) for _ in control]
    phases += [("data", 0) for _ in data_phase]
    phases += [("nominal", 0) for _ in tail]

    seq = control + data_phase + tail
    t = 0.0
    for (kind, _), b in zip(phases, seq):
        dt = t_nominal_us if kind == "nominal" else t_data_us
        events.append((t, t + dt, kind, b))
        t += dt
    return events
```

打印每个 bit 时长，直观看到 BRS 之后数据段明显"更窄更快"，最后 ACK/EOF 又回到 nominal：

```python
control, data_phase, tail = canfd_data_frame(0x100, bytes(8), brs=True)
tl = fd_timeline(control, data_phase, tail)

print("段 | 时长(us) | 电平")
for count, (start, end, kind, b) in enumerate(tl):
    print(f"{kind:8s} | {end-start:.2f} | {b}")
    if count > 28:
        break
```

输出（示意）：前面接近 `nominal 2.00us`，BRS 后数据段 `data 0.20us`，再回到 `nominal 2.00us`。

---

## 🆚 9. 汇总对比（模拟里体现的差异）

| 项目 | 普通 CAN | CAN-FD |
| :-- | :-- | :-- |
| `brs` | 无 | 有，数据段位时长变短 |
| 数据长度 | ≤ 8 字节 | ≤ 64 字节，`DLC 9~15` 映射 |
| CRC | `crc15_can` | `crc17_fd` / `crc21_fd` |
| 位填充 | 经典规则 | 数据段另有规则（此处不展开） |
| 远程帧 | 支持 | 不支持 |

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 运行本模拟代码前必过
> - [ ] **电平映射**：显性=逻辑 0=优先级高，勿与直觉混淆
> - [ ] **位填充往返**：`unstuff(stamp(x)) == x` 自检通过
> - [ ] **CRC 范围**：SOF 到数据段末尾（不含分隔符）
> - [ ] **CRC 选择**：FD ≤16B 用 CRC-17，>16B 用 CRC-21
> - [ ] **仲裁模型**：发隐性读到显性即失败退出，ID 小者胜
> - [ ] **解码对齐**：先剥固定尾部 → 去填充 → 按 DLC 取数据
> - [ ] **DLC 映射**：FD 长度 9~64 用 `fd_dlc_for_length`
> - [ ] **双速率**：数据段用 `t_data_us`，ACK/EOF 回 `t_nominal_us`
> - [ ] **参数验证**：真实接入前用目标控制器交叉验证 CRC

---

## 🔗 思考与延伸索引

- [[CAN]] — CAN / CAN-FD 协议层总览：物理层、仲裁、帧格式、错误状态机
- [[Modbus-code]] — 另一种协议的 Python 软件模拟对照
- [[SPI-code]] — SPI 的软件模拟实现
- [[UART-code]] — UART 帧编解码的软件模拟
- [[嵌入式内核]] — 真实 CAN 控制器驱动中的位时序与错误处理

---

## 📚 参考资料

1. **Bosch** — *CAN Specification Version 2.0*, 1991.
2. **Bosch** — *CAN with Flexible Data-Rate Specification Version 1.2*, 2012.
3. **ISO** — *ISO 11898-1: CAN — Data link layer and physical signalling*.
4. **Python** — `python-can`、`cantools` 官方文档.
