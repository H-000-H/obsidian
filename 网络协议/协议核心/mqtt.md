---
title: MQTT 协议完整技术手册
tags:
  - Network
  - Protocol
  - MQTT
  - IoT
status: Completed
---

# 📡 MQTT 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：运行在 TCP 之上的轻量级**发布/订阅（Publish/Subscribe）**消息协议，由 **Broker（消息代理）** 集中路由，生产者与消费者在**空间、时间、同步**三个维度完全解耦；控制报文（Control Packet）报头最小 **2 字节**，为低带宽、高时延、不可靠网络与资源受限终端而设计。
> - **内容范围**：协议定位 · 主题体系与通配符 · 报文结构（固定报头 / 剩余长度变长编码） · 14 种控制报文类型 · CONNECT/CONNACK · QoS 0/1/2 完整交互 · Keep Alive 与遗嘱 · 保留消息 / 持久会话 / 共享订阅 · 实现与排错
> - **推荐路径**：§1 定位 → §2 主题 → §3 报文结构（核心） → §4 类型总表 → §5 连接 → §6 QoS（核心） → §7 可靠性特性 → §8 排错

---

## 📡 1. 协议定位与体系结构

### 1.1 三种角色

MQTT 是**客户端-服务器**架构，但服务器（Broker）承担路由职责，端点彼此不直连：

| 角色 | 英文 | 职责 | 是否知道对方存在 |
| :--- | :--- | :--- | :--- |
| **发布端** | Publisher | 向某**主题（Topic）** 发送应用消息 | 否，只认识主题 |
| **代理服务端** | Broker / Server | 维护连接、匹配订阅、路由与存储转发、维持会话与保留消息 | 认识所有客户端 |
| **订阅端** | Subscriber | 用**主题过滤器（Topic Filter）**声明兴趣，接收消息 | 否，只认识主题 |

### 1.2 解耦的四个维度

| 解耦类型 | 含义 | 协议机制 |
| :--- | :--- | :--- |
| **空间解耦** | 收发双方无需知道对方网络地址 | 以主题为中介（Content-based routing） |
| **时间解耦** | 双方无需同时在线 | 持久会话 + Broker 队列 + 保留消息 |
| **同步解耦** | 发布是"发射即忘"，不阻塞等待消费者 | QoS 由 Broker 分担，非端到端同步 |
| **流量解耦** | 一对多扇出（Fan-out）不需要 N 次连接 | 一次发布，多订阅者各自按自身 QoS 接收 |

> [!NOTE] 代价
> 松耦合把**状态搬到了 Broker**：会话队列、遗嘱、保留消息、待确认（in-flight）报文都需要服务端持久化，Broker 因此成为可用性单点，需集群/热备（见 §8.2）。

### 1.3 传输层与端口

MQTT 自身不实现可靠传输，**可靠字节流交给 TCP**（见 [[TCP]]），协议只负责应用层语义：

| 传输 | 默认端口 | 说明 |
| :--- | :---: | :--- |
| **TCP（明文）** | **1883** | IANA 注册的 MQTT 默认端口 |
| **TLS over TCP** | **8883** | 1883 + TLS，常用单向/双向证书（见 [[TLS]]） |
| **WebSocket** | 9001 / 8083（wss 8084 / 443） | 浏览器与穿透 HTTP 代理场景，按实现而异 |
| **MQTT-SN / CoAP 等** | — | 串口、BLE、802.15.4 等非 TCP 承载，另有规范 |

> [!WARNING] MQTT 不承载于 UDP
> 标准 MQTT 只跑在**有序、可靠**的字节流上。需要 UDP 语义时属于另一协议族（MQTT-SN、CoAP），不要自行把 MQTT 报文裸发到 UDP 上。

### 1.4 MQTT vs HTTP

| 维度 | **MQTT** | **HTTP** |
| :--- | :--- | :--- |
| 交互模型 | 发布/订阅，异步、面向消息 | 请求/响应，同步、面向资源 |
| 连接方式 | **长连接**（一次 CONNECT，长期复用） | 短连接或 Keep-Alive 复用，请求驱动 |
| 最小头部开销 | **2 字节**（固定报头） | 数百字节 ~ KB 级文本首部 |
| 推送能力 | Broker 可主动下行 PUBLISH | 服务端不能主动推（需轮询/SSE/WebSocket） |
| 状态 | 会话状态在服务端保持（订阅、遗嘱、保留、队列） | 无状态，靠 Cookie/Token 外挂 |
| 可靠性语义 | 协议内置 **QoS 0/1/2** | 传输层可靠 + 应用层自行重试 |
| 弱网/NAT | 需 **Keep Alive 心跳**保活与断线检测 | 请求天然穿透，适配性好 |
| 典型场景 | 遥测、传感、指令下发、百万级设备接入 | 文档、REST API、Web 前端 |

### 1.5 协议版本与嵌入式适用性

| 版本 | Protocol Level | 关键特征 |
| :--- | :--: | :--- |
| MQTT **v3.1** | 3 | IBM 首发（2010），报文标识符长度 1 字节（遗留问题） |
| MQTT **v3.1.1** | 4 | **OASIS 标准（2014）**，标识符扩为 2 字节，本手册主线 |
| MQTT **v5.0** | 5 | 增强错误码 Reason Code、Properties、共享订阅、主题别名、消息过期间隔 |

报文头最小 2 字节、二进制紧凑编码、无需文本解析这三点，使 Paho Embedded、NanoMQ 等实现可以在 RAM 仅数十 KB 的 MCU 上工作：组包与解析全程用 `etl::vector<uint8_t, N>` / `etl::buffer` 之类的**定容无堆缓冲**预分配，配合 `pre_execution()` 一次性建立连接，即可在 mini_tree 上层业务里做到零运行时 `malloc`（详见 [[UART]] 所在仓库的分层约定）。

---

## 🏷️ 2. 主题体系（Topic Space）

### 2.1 主题命名规则

| 规则 | 说明 |
| :--- | :--- |
| **层级分隔符** | `/`（斜杠），如 `factory/line3/plc12/temp` |
| **字符集** | UTF-8 编码；主题名与过滤器区分**大小写**（case-sensitive） |
| **长度上限** | 单级无限制，整体最长 $2^{16}-1 = 65535$ **字节**（含 2 字节长度前缀） |
| **空层级** | `a//b` 合法，中间空层级会被 `+` 匹配 |
| **通配符** | `+` 与 `#` **只允许出现在订阅过滤器中**，发布时使用即为协议错误 |
| **`$` 前缀** | 以 `$` 开头的主题为协议/保留用途（如 `$SYS`），通配符订阅默认**不匹配**首层 `$` |

### 2.2 通配符（Wildcard）

| 通配符 | 名称 | 匹配范围 | 约束 |
| :--: | :--- | :--- | :--- |
| **`+`** | 单层通配符（Single-Level） | 恰好**一个**层级（可为空层级） | 必须**独占一层**：`a/+` ✔　`a/+b` ✘ |
| **`#`** | 多层通配符（Multi-Level） | **零个或多个**后续层级 | 必须位于**末尾且独占一层**：`a/#` ✔　`a#` ✘　`a/#/b` ✘ |

匹配示例（过滤器 `home/+/temperature`）：

| 主题名 | 是否匹配 | 原因 |
| :--- | :--: | :--- |
| `home/kitchen/temperature` | ✔ | `+` 匹配 `kitchen` |
| `home/temperature` | ✘ | 层级数不符（`+` 必须占一层） |
| `home/a/b/temperature` | ✘ | `+` 不能跨层 |
| `home//temperature` | ✔ | `+` 可匹配空层级 |

| 过滤器 | 匹配 | 不匹配 |
| :--- | :--- | :--- |
| `home/#` | `home`、`home/a`、`home/a/b/c` | `office/a`、`$SYS/...` |
| `#` | 所有**不以 `$` 开头**的主题 | `$SYS/broker/...` |
| `$SYS/#` | 所有系统主题（需显式写出 `$`） | — |

### 2.3 `$SYS` 系统主题

Broker 通过 `$SYS/` 前缀发布自身运行指标，客户端**只能订阅、不能发布**，且通常需要在 CONNECT 之前或配置里显式授权：

```text
$SYS/broker/clients/connected     当前在线客户端数
$SYS/broker/clients/total         累计/已注册客户端数（含持久会话离线）
$SYS/broker/messages/received     累计接收报文数
$SYS/broker/messages/sent         累计发送报文数
$SYS/broker/load/messages/sec/1min  近 1 分钟消息速率（滑动平均）
$SYS/broker/uptime                运行时长
```

### 2.4 主题设计建议

> [!TIP] 主题分层实践
> 1. **由粗到细**：`{业务域}/{站点}/{设备}/{指标}`，把选择性最强的层级放在靠后位置，便于 Broker 前缀树剪枝。
> 2. **不要把值当层级**：`temp/23` 会产生无限主题爆炸，值应放进**载荷**，层级只放"身份"。
> 3. **深度可控**：过深会增加匹配开销与通配符歧义，一般 3~5 级足够。
> 4. **预留运维分支**：`.../status`（遗嘱/在线离线）、`.../cmd`（下行）、`.../ota`，让监控与业务主题天然分离。
> 5. **ACL 与主题同构**：`{org}/{site}/#` 形式的鉴权规则最省事，避免用正则做主题授权。

---

## 🧱 3. 报文结构（核心）

### 3.1 三段式总览

所有 MQTT 控制报文都是同一种骨架：

```text
┌────────────────────┬──────────────────┬────────────────────┐
│ 固定报头 Fixed     │ 可变报头 Variable│ 有效载荷 Payload   │
│ Header (2~6 B)     │ Header (可选)    │ Header (可选)      │
├────────────────────┼──────────────────┼────────────────────┤
│ 字节1:             │ 报文标识符 2 B   │ Client ID          │
│  类型 4bit│标志4bit│ (CONNECT/PINGREQ │ Topic Name         │
│ 字节2~5:           │  /DISCONNECT 等  │ Packet Identifier  │
│  剩余长度 1~4 B    │ 无此字段)        │ Will Topic/Message │
│                    │                  │ User Name/Password │
└────────────────────┴──────────────────┴────────────────────┘
        剩余长度 = 可变报头字节数 + 有效载荷字节数
```

### 3.2 固定报头（Fixed Header）

**第 1 字节**：高 4 位 = 控制报文类型（Control Packet Type），低 4 位 = 标志位（Flags）。

```text
 7   6   5   4   3   2   1   0
├───────────────┼───────────────┤
│ Type (4 bit)  │ Flags (4 bit) │   ← 第 1 字节
└───────────────┴───────────────┘
```

**PUBLISH 的标志位最有代表性**（其余多数报文标志位固定为 `0000`）：

```text
      7   6   5 │ 4   3 │ 2   1 │ 0
PUBLISH  0  0  1  1 │ DUP │QoS│QoS│ RETAIN
                    0  0  1  1   └ 0x01 Retain   └ 0x02/0x04 QoS   └ 0x08 DUP
```

各类型第 1 字节的常见取值：

| 报文 | 第 1 字节 | 说明 |
| :--- | :--: | :--- |
| CONNECT | `0x10` | 类型 1，标志必须为 `0000` |
| CONNACK | `0x20` | 类型 2，标志必须为 `0000` |
| PUBLISH | `0x30` \| (DUP<<3) \| (QoS<<1) \| RETAIN | 唯一大量使用标志位者 |
| PUBACK / PUBREC / PUBCOMP | `0x40` / `0x50` / `0x70` | 标志必须为 `0000` |
| **PUBREL** | **`0x62`** | **标志必须为 `0010`**，收到 `0x60` 视为协议违规 |
| **SUBSCRIBE / UNSUBSCRIBE** | **`0x82` / `0xA2`** | **标志必须为 `0010`** |
| SUBACK / UNSUBACK | `0x90` / `0xB0` | 标志必须为 `0000` |
| PINGREQ / PINGRESP | `0xC0` / `0xD0` | 无载荷 |
| DISCONNECT | `0xE0` | 正常断开 |

> [!CAUTION] 标志位是硬性校验项
> v3.1.1 规定：若某类报文的保留标志位未按要求置位，接收方**必须断开网络连接**（Malformed Packet）。排错时抓到的"连上就被踢"多半源于此，而非鉴权失败。

### 3.3 剩余长度（Remaining Length）变长编码

紧跟第 1 字节的是 **1~4 字节**的剩余长度字段，采用"每字节 7 bit 有效数据 + 1 bit 续位（continuation bit）"的 **变长编码（Variable Byte Integer）**。

- **bit7 = 1**：后面还有长度字节；**bit7 = 0**：长度字段到此结束。
- 低位在前：第 1 个字节放最低 7 位，依次向左。
- 编码范围：$0 \sim 268{,}435{,}455$ 字节（≈256 MB），4 字节即为上限。
- 值 = $\sum_{i=0}^{n} (\text{byte}_i \bmod 128) \times 128^i$

**编码算法**（C 风格伪码，可直接落到 MCU）：

```c
// 编码：先取低 7 位，只要还有高位剩余就把续位置 1
do
{
    uint8_t encoded = (uint8_t)(length % 128);
    length /= 128;
    if (length > 0)
    {
        encoded |= 0x80;      /* 还有后续字节 */
    }
    buf[i++] = encoded;
} while (length > 0);
```

**示例：剩余长度 = 321 字节**

| 步骤 | 计算 | 结果 | 字节值 |
| :--: | :--- | :--- | :--: |
| 1 | $321 \bmod 128 = 65$，$321 / 128 = 2 > 0$ → 置续位 | $65 \vert 0\text{x}80$ | **`0xC1`** |
| 2 | $2 \bmod 128 = 2$，$2 / 128 = 0$ → 续位为 0 | $2$ | **`0x02`** |
| 校验 | $0\text{x}41 + 0\text{x}02 \times 128 = 65 + 256$ | $= 321$ ✔ | `C1 02` |

对照：一个"载荷 321 字节的 QoS0 非保留 PUBLISH"，其第 1 字节为 `0x30`，完整前缀为 `30 C1 02 …`（**仅 3 字节开销**）。

**127 边界与进位表**：

| 剩余长度值 | 字节数 | 编码 | 说明 |
| :--- | :--: | :--- | :--- |
| $0$ | 1 | `0x00` | 如 PINGREQ / DISCONNECT |
| **$127$（$2^7-1$）** | **1** | **`0x7F`** | **1 字节上限** |
| **$128$（$2^7$）** | **2** | **`0x80 0x01`** | **进位：首个跨界值** |
| $16{,}383$（$2^{14}-1$） | 2 | `0xFF 0x7F` | 2 字节上限 |
| $16{,}384$ | 3 | `0x80 0x80 0x01` | |
| $2{,}097{,}151$ | 3 | `0xFF 0xFF 0x7F` | 3 字节上限 |
| $2{,}097{,}152$ | 4 | `0x80 0x80 0x80 0x01` | |
| $268{,}435{,}455$ | 4 | `0xFF 0xFF 0xFF 0x7F` | 协议最大值 |

> [!TIP] 实现陷阱
> 解码循环必须**边读边判断是否超过 4 字节**，否则畸形报文可让解析器无限读取；同时应对"声称的长度"与"实际可读字节数"做上限校验，防止按长度字段预分配巨大缓冲（DoS 面）。

### 3.4 可变报头与有效载荷

| 段 | 内容 | 何时存在 |
| :--- | :--- | :--- |
| **可变报头** | 报文标识符（2 B）、CONNECT 的协议名/级别/标志/Keep Alive、CONNACK 的确认标志/返回码、SUBACK 的返回码列表 | QoS 0 的 PUBLISH 无标识符；CONNECT/PINGREQ/PINGRESP/DISCONNECT/CONNACK 无"报文标识符" |
| **有效载荷** | Topic Name、应用消息、Client ID / Will / 用户名口令、主题过滤器列表 | 由固定报头的剩余长度界定 |

**报文标识符（Packet Identifier）**：16 bit，$1 \sim 65535$（0 不使用），由**发送方**为每个需要确认的报文（QoS>0 的 PUBLISH、SUBSCRIBE/SUBACK、UNSUBSCRIBE/UNSUBACK）分配，用于把应答与请求配对；同一标识符在**上一个交互未完成前不得复用**。

---

## 📇 4. 控制报文类型总表（14 种）

| 类型值 | 报文 | 标志位要求 | 方向 | 报文标识符 | 作用与 QoS 关联 |
| :--: | :--- | :--: | :--- | :--: | :--- |
| `0` / `15` | *Reserved* | — | — | — | **保留值，收到即协议错误并断开** |
| **1** | **CONNECT** | `0000` | 客户端 → 服务端 | 无 | 建立连接，携带协议级别、Clean Session、Keep Alive、Will、凭据 |
| **2** | **CONNACK** | `0000` | 服务端 → 客户端 | 无 | 连接应答：Session Present + 返回码 |
| **3** | **PUBLISH** | DUP/QoS/Retain | 双向 | QoS 1、2 必带 | 业务数据载体，唯一携带 QoS 位的数据报文 |
| **4** | **PUBACK** | `0000` | 双向 | 必带 | **QoS 1** 的确认（第 2 步，终点） |
| **5** | **PUBREC** | `0000` | 双向 | 必带 | **QoS 2** 第 2 步："已收到并去重存放" |
| **6** | **PUBREL** | `0010` | 双向 | 必带 | **QoS 2** 第 3 步："可以释放了" |
| **7** | **PUBCOMP** | `0000` | 双向 | 必带 | **QoS 2** 第 4 步："交付完成" |
| **8** | **SUBSCRIBE** | `0010` | 客户端 → 服务端 | 必带 | 声明主题过滤器 + 请求 QoS（0/1/2） |
| **9** | **SUBACK** | `0000` | 服务端 → 客户端 | 必带（同请求） | 逐条返回授予的 QoS 或 `0x80` 失败 |
| **10** | **UNSUBSCRIBE** | `0010` | 客户端 → 服务端 | 必带 | 取消订阅（不要求载荷 QoS 字段） |
| **11** | **UNSUBACK** | `0000` | 服务端 → 客户端 | 必带（同请求） | 取消完成，无载荷 |
| **12** | **PINGREQ** | `0000` | 客户端 → 服务端 | 无 | 心跳请求，零载荷 |
| **13** | **PINGRESP** | `0000` | 服务端 → 客户端 | 无 | 心跳应答，零载荷 |
| **14** | **DISCONNECT** | `0000` | 客户端 → 服务端（v5 双向） | 无 | **有序断开**：服务端不应发布遗嘱 |

---

## 🔌 5. CONNECT 与 CONNACK 字段详解

### 5.1 CONNECT 可变报头

| 偏移 | 字段 | 长度 | 说明 |
| :--: | :--- | :--: | :--- |
| 0~1 | **Protocol Name** | 4 B | `0x00 0x04 'M' 'Q' 'T' 'T'`（v3.1 同名，v5.0 亦为 `MQTT`） |
| 2 | **Protocol Level** | 1 B | `3` = v3.1，**`4` = v3.1.1**，`5` = v5.0，其余非法 |
| 3 | **Connect Flags** | 1 B | 见 §5.2 |
| 4~5 | **Keep Alive** | 2 B | 单位**秒**，`0x0000` 表示禁用心跳（不推荐） |

### 5.2 Connect Flags 位域

```text
 7        6        5        4   3        2         1         0
Reserved Clean  Will   Will QoS(2bit)  Will   Password User
  (0)    Session Flag               Retain   (1bit)   Name
```

| 位 | 名称 | 含义 |
| :--: | :--- | :--- |
| 7 | **Reserved** | 必须为 `0`，否则协议错误 |
| 6 | **Clean Session** | `1`：丢弃旧会话、以新会话开始；`0`：**持久会话**，请 Broker 找回订阅与排队消息 |
| 5 | **Will Flag** | `1`：本次连接注册遗嘱消息，异常断开时由 Broker 代发 |
| 4-3 | **Will QoS** | 遗嘱发布 QoS：`0`/`1`/`2`，`3` 保留非法 |
| 2 | **Will Retain** | 遗嘱是否作为**保留消息**发布 |
| 1 | **Password** | `1` → 载荷含 Password 字段 |
| 0 | **User Name** | `1` → 载荷含 User Name 字段 |

> [!WARNING] 顺序耦合
> 口令与用户名是"标志位决定存在与否 + 载荷按固定顺序出现"，没有字段名，纯靠偏移解析。标志位与实际载荷字段数量不一致（多写/漏写）是自制客户端最常见的连不上原因。

### 5.3 遗嘱三要素（Will）

遗嘱由三部分组成，全部在 **CONNECT 时**注册、在**客户端异常断开时**由 Broker 代发：

| 要素 | 字段 | 说明 |
| :--- | :--- | :--- |
| **话题** | Will Topic | 遗嘱发布到的主题，如 `factory/line3/plc12/status` |
| **内容** | Will Message | 载荷，如 `offline`、错误码或 JSON |
| **发布属性** | Will QoS + Will Retain | 决定可靠性与是否被新订阅者立即拿到 |

**触发条件**：TCP 非正常关闭、Keep Alive 超时被判定失联 → **触发**；客户端主动发 `DISCONNECT` → **不触发**（v3.1.1 语义）。因此"遗嘱 + 上线时主动发布 `online`"才是完整的在线状态方案（Online LWT 模式）。

### 5.4 CONNECT 有效载荷字段顺序

```text
Client ID → [Will Topic] → [Will Message] → [Password] → [User Name]
   必有        Will Flag=1    Will Flag=1     Flag bit1     Flag bit0
```

- **Client ID**：$1 \sim 65535$ 字节；服务端**至少**要接受 1~23 字节内的字母数字；同一 Broker 下重复的 Client ID 会**顶掉旧连接**（表现为两设备互踢、日志里 `duplicate clientid`）。
- 用户名/口令：**明文**随 CONNECT 发送，生产环境必须走 8883/TLS 或服务端注入证书（见 [[TLS]]）。

### 5.5 CONNACK

| 字段 | 长度 | 说明 |
| :--- | :--: | :--- |
| **确认标志（Acknowledgement Flags）** | 1 B | bit0 = **Session Present**：`1` 表示服务端恢复了旧的持久会话（订阅/队列已续用）；bit1~6 保留为 0 |
| **返回码（Return Code）** | 1 B | 见下表 |

| 返回码 | 含义 | 排查方向 |
| :--: | :--- | :--- |
| `0x00` | **Connection Accepted** | — |
| `0x01` | Unacceptable protocol version | `--protocol-version` 与 Broker 支持级别不符 |
| `0x02` | Identifier rejected | Client ID 非法/超长/被 ACL 拒绝 |
| `0x03` | Server unavailable | Broker 未就绪、连接数打满、正在重启 |
| `0x04` | Bad user name or password | 凭据错误（口令未启用时**不得**同时携带用户名） |
| `0x05` | Not authorized | 账号存在但无权限 / ACL 未放行 |

> [!NOTE] v5.0 变化
> v5.0 把 6 个返回码扩为**双字节 Reason Code（0x00~0x97）**并引入 Properties（如 `Server Keep Maximum`、`Topic Alias Maximum`、`Reason String`），排错时能拿到"为什么被拒"的文本描述。

---

## 🎚️ 6. QoS 0 / 1 / 2：三级服务质量（核心）

### 6.1 语义定义

| QoS | 名称 | 交付语义 | 交互报文数 | 适用 |
| :--: | :--- | :--- | :--: | :--- |
| **0** | At most once（至多一次） | 发一次即忘，**可能丢，不重复** | 1 | 高频遥测（1 Hz 温度），丢一帧无妨 |
| **1** | At least once（至少一次） | **不丢，可能重复** | 2 | 一般业务上报，接收端按 id 幂等去重 |
| **2** | Exactly once（恰好一次） | **不丢且不重复** | 4 | 计费、告警、指令等不可重放的关键消息 |

### 6.2 QoS 0：Fire and Forget

```text
Publisher ──── PUBLISH (QoS=0, 无报文标识符) ────► Broker ──── PUBLISH ────► Subscriber
                     （到此结束，没有任何确认）
```

- 报文**没有** Packet Identifier（省 2 字节），一旦交给 TCP 就认为发送完成。
- Broker 到订阅者的转发独立按**订阅者自己的 QoS** 决定，与上游无关。

### 6.3 QoS 1：两次握手 + 重传

```text
时刻  Publisher                          Broker
 t0  │ ── PUBLISH  (QoS=1, DUP=0, id=7) ─►│  存储/转发，入队
 t1  │ ◄─ PUBACK   (id=7) ────────────────│  确认到达
─────┴── 若在超时前未收到 PUBACK：
 t0' │ ── PUBLISH  (QoS=1, DUP=1, id=7) ─►│  同一 id 重发，Broker 侧去重
 t1' │ ◄─ PUBACK   (id=7) ────────────────│
─────┴────────────────────────────────────┴──────────────────────
下行：Broker ── PUBLISH(QoS=1) ──► Subscriber ── PUBACK ──► Broker
```

要点：

1. **DUP 位**：`DUP=1` 表示"这是一个**重传**"，只在 PUBLISH 中出现；v3.1.1 允许 QoS 1/2 都置位，**v5.0 规定仅 QoS 1 可置位**（QoS 2 的状态机已能表达重发，置位反而造成歧义）。
2. **重复来源**：PUBACK 丢失、或重传 PUBLISH 与原始 PUBLISH 均被处理 → 订阅者可能收到**两条相同内容**，**去重责任在应用层**（按 id 或业务幂等键）。
3. **发送端状态**：报文进入"未确认（in-flight / QoS1 等待 PUBACK）"集合，需**持久化**才能跨重连恢复。

### 6.4 QoS 2：四次握手（PUBREC / PUBREL / PUBCOMP）

```text
时刻   Publisher                             Broker
 t0    │ ── PUBLISH (QoS=2, DUP=0, id=7) ───►│ 状态：等待 PUBREL
 t1    │ ◄─ PUBREC  (id=7) ──────────────────│ 已落盘去重，暂不转发
 t2    │ ── PUBREL  (id=7, flags=0010) ────►│ 状态：等待 PUBCOMP
 t3    │ ◄─ PUBCOMP (id=7) ──────────────────│ 交付完成，解除存储
───────┴─────────────────────────────────────┴──────────────────────────
发布端状态机：  等待 PUBREC ──(收到 PUBREC)──► 等待 PUBCOMP ──(收到 PUBCOMP)──► 空闲
接收端状态机：  等待 PUBREL ──(收到 PUBREL)──► 等待 PUBCOMP ──(收到 PUBCOMP)──► 空闲
```

- **为什么需要四步**：QoS 1 的"一个 ACK"无法区分"我没收到"与"我收到了但你的 ACK 丢了"。QoS 2 用**两级确认 + 释放**把"去重存储"和"投递放行"分开：`PUBREC` 表示"我已可靠保存并记住了这个 id"，`PUBREL` 表示"我可以释放它、你不必再重发"，`PUBCOMP` 表示"握手结束"。
- **恰好一次如何实现**：接收端在**未收到 PUBREL 前不会把消息交给上层**，收到后按 id 丢弃后续重复 PUBLISH（Broker 侧同样以 id 去重表拦截），因此端到端只交付一次。
- **会话状态保持**：QoS 2 的握手可以**跨断线重连继续**（依赖 Clean Session=0 的持久会话与 id 持久化），若中途彻底丢失会话状态，只能降级为"至多一次"。
- **代价**：报文数翻倍、往返时延约为 QoS 1 的两倍、每 id 需要额外存储；下行时 Broker 还要为每个订阅者各自跑一遍完整四次握手。

### 6.5 三者开销对比

| 维度 | **QoS 0** | **QoS 1** | **QoS 2** |
| :--- | :--- | :--- | :--- |
| 报文数（单方向） | 1 | 2 | 4 |
| 网络往返（RTT） | 0（不等待） | 1 | 2 |
| 报文标识符 | 无 | 需要 | 需要 |
| 发送端存储 | 无 | 未确认报文 | 未确认报文 + 握手状态 |
| 接收端去重表 | 无 | 可选（应用层） | **必需**（协议层） |
| 消息丢失风险 | **有** | 无 | 无 |
| 消息重复风险 | 有（重连后可能重复） | **有** | 无 |
| 头部/状态开销 | 最低 | 中 | 高（约 QoS0 的 2 倍字节 + 存储） |
| 典型场景 | 秒级遥测、心跳 | 状态上报、事件 | 计费、指令、告警 |

> [!IMPORTANT] QoS 不升级、只可能降级
> 端到端 QoS = **min(发布时 QoS, 订阅时 QoS)**。QoS 2 发布 + QoS 0 订阅 = 订阅者只拿到 QoS 0，且**无法**由 Broker 提升。订阅者收到的重复消息也不能由协议替你吞掉。

---

## 💓 7. 保活、遗嘱与消息属性

### 7.1 Keep Alive 心跳与断线检测

- 客户端承诺：相邻两个**发出**的报文间隔不超过 `Keep Alive` 秒；空闲达阈值时发 **PINGREQ**，服务端**必须**回 **PINGRESP**（服务端也可主动下发任意报文来充当心跳）。
- **1.5 倍规则**：服务端在 $1.5 \times \text{Keep Alive}$ 时间内未收到任何报文，**必须**断开连接，并按需发布该客户端的遗嘱。

$$
T_{\text{detect}} = 1.5 \times T_{\text{keepalive}}
$$

| Keep Alive | 检测超时 | 建议 |
| :--: | :--: | :--- |
| 30 s | 45 s | 移动网络/公网终端常用，能快速发现假连接 |
| 60 s | 90 s | 平衡带宽与功耗 |
| 120 s | 180 s | 有线内网、流量敏感场景 |

> [!CAUTION] 心跳与 NAT/负载均衡超时
> 家用 NAT、云 LB 的 UDP/TCP 空闲超时常见 **60~300 s**，一旦空闲即静默丢连接而客户端不自知。经验法则：**Keep Alive 取 NAT/LB 空闲超时的 1/2 至 1/3**，并且**客户端侧**也要做"未收到 PINGRESP 即判定失联并重连"的检测——不要只依赖服务端超时。

### 7.2 遗嘱消息（LWT, Last Will and Testament）

见 §5.3。典型在线状态方案：

```text
CONNECT: Will Topic=f/.../status  Will Message=offline  Will QoS=1  Will Retain=1
上线后立刻: PUBLISH f/.../status "online" (retain=1, QoS=1)
正常退出:    DISCONNECT（Broker 不发遗嘱，可再主动发一条 "offline"）
异常掉线:    Broker 在 1.5×KeepAlive 后代发 "offline"
```

### 7.3 保留消息（Retained）

| 行为 | 说明 |
| :--- | :--- |
| 设置 | PUBLISH 的 `RETAIN=1`，Broker **按主题只保存最新一条** |
| 投递 | 订阅者**一订阅就立刻**收到该保留消息（无需等下一次发布） |
| 清除 | 向同一主题发布 **`RETAIN=1` 且载荷为 0 字节** 的消息（`mosquitto_pub -r -t X -n`） |
| 标志 | 转发保留消息时 `RETAIN=1`，供接收端识别"这是历史快照，不是新事件" |

适用：设备状态、当前配置值。不适合：高频遥测（每条都覆盖存储、且新订阅者会拿到毫无意义的瞬时值）。

### 7.4 持久会话（Persistent / Durable Session）

| Clean Session | 断线后 Broker 行为 |
| :--: | :--- |
| **1** | 连接关闭即**清除**会话：丢弃订阅关系与排队消息，重连后必须重新 SUBSCRIBE |
| **0** | 按 **Client ID** 保留会话：维持订阅、继续为 QoS 1/2 排队消息（受队列上限与存储策略约束）、完成中断的握手、代发遗嘱 |

- v5.0 将其拆为 **Clean Start**（是否清除旧会话）+ **Session Expiry Interval**（会话保留多久，`0xFFFFFFFF` 为永不过期），比 0/1 二值更精细。
- 持久会话不等于"消息一定不丢"：Broker 重启、队列溢出（超过 `max_queued_messages`）仍会丢弃，需结合磁盘持久化（EMQX/mosquitto persistence 文件）。

### 7.5 共享订阅与 v5.0 其他特性（简述）

| 特性 | 语法 | 说明 |
| :--- | :--- | :--- |
| **共享订阅**（v5.0） | `$share/{Group}/{Filter}` | 同组多个消费者**负载均衡**分发，一条消息只投给组内一个成员；解决"一个消费组横向扩容"，注意通配符不能紧跟 `$` 首层 |
| 主题别名 | Property: `Topic Alias` | 用 2 字节整数代替长主题，高频上行可显著省带宽 |
| 消息过期间隔 | Property: `Message Expiry Interval` | 队列中过期即丢弃，避免持久会话积压成"过期数据洪水" |
| 请求/响应 | `Response Topic` + `Correlation Data` | 在 Pub/Sub 上做 RPC 语义 |
| Reason Code / Properties | 各应答报文 | 排错可读性大幅提升 |

---

## 🛠️ 8. 实现与排错

### 8.1 mosquitto 常用命令

```bash
# ── Broker：前台启动 + 详细日志（调试首选）
mosquitto -v -p 1883
mosquitto -c /etc/mosquitto/mosquitto.conf          # 生产：配置文件（含监听/认证/acl）

# ── 订阅
mosquitto_sub -h 127.0.0.1 -p 1883 -t 'factory/line3/#' -q 1 -v        # -v 打印主题前缀
mosquitto_sub -L ssl://broker.example.com:8883 --cafile ca.crt \
              -u dev -P 'secret' -t '$SYS/broker/clients/connected' -i mon-01
mosquitto_sub -V mqttv5 -t '$share/grpA/sensor/#' -q 2 -d              # 共享订阅 + 调试帧
mosquitto_sub -t 'f/#' --retained-only                                 # 只看保留消息快照

# ── 发布
mosquitto_pub -t 'factory/line3/plc12/status' -m 'online' -q 1 -r      # 设置保留消息
mosquitto_pub -t 'factory/line3/plc12/status' -r -n                    # 0 字节载荷 → 清除保留
mosquitto_pub -t 'dev/cmd' -m 'reboot' -q 2 --id-prefix ota            # QoS 2
mosquitto_pub -t 'sensor/temp' -m '' -W 100                            # 100 ms 间隔压测

# ── 认证与抓包
mosquitto_passwd -c /etc/mosquitto/passwd dev                          # 生成口令文件
tcpdump -i any -nn -s0 -A 'tcp port 1883'                              # 明文口直接看报文字节
mosquitto_sub ... -d            # -d 打印收到的原始报文（hex dump），核对类型/DUP/QoS 位
```

### 8.2 Broker 选型一句话

**mosquitto**：单二进制、KB 级内存、可跑在网关与树莓派上，单机万级连接内首选；**EMQX**：分布式集群、百万级并发、规则引擎/桥接（Kafka、DB）与 Dashboard（18083），平台化生产首选。

> [!NOTE] 中间件对照
> Kafka/RabbitMQ 是"队列/日志总线"语义，面向服务端集群；MQTT 是"受限设备 + 不可靠网络"语义，面向海量长连接。实际架构常见组合：**MQTT 接入层 → Kafka 骨干 → 消费者集群**。

### 8.3 常见坑

| 坑 | 现象 | 处置 |
| :--- | :--- | :--- |
| **主题大小写不一致** | 订阅 `Sensor/Temp` 收不到 `sensor/temp` | 主题严格区分大小写；统一命名规范并用 `$share`/ACL 校验工具检查 |
| 发布时用了通配符 | 连接被断开或消息丢弃 | `+`/`#` 仅用于订阅过滤器 |
| `#` 写法非法 | 订阅不生效 | 必须独占末层：`a/#` ✔、`a#` ✘ |
| 收不到 `$SYS` | `#` 订阅也无数据 | 需显式订阅 `$SYS/#`，部分 Broker 要求配置开启 |
| **重连风暴**（Reconnect Storm） | Broker 重启后百万设备同时重连，CPU/TLS 握手打满，反复雪崩 | **指数退避 + 随机抖动（jitter）**；错峰重连；服务端限流（连接速率阈值） |
| **心跳与 NAT 超时** | "看似在线、实际收不到消息"，或每隔固定几分钟掉线 | Keep Alive 设为 NAT/LB 空闲超时的 1/2~1/3；客户端也检测 PINGRESP |
| 重复 Client ID | 两台设备互相顶掉，日志 `duplicate clientid` | Client ID 全局唯一（用 SN/MAC），或用 `${clientid}` 占位与网关注册 |
| QoS 1 重复上报 | 同一业务事件处理两次 | 应用层按 id/幂等键去重，或改用 QoS 2 |
| 持久会话队列爆 | 内存持续增长、订阅延迟巨大 | 限制队列深度 + 丢弃策略；v5 用 Message Expiry |
| 大报文 + 低带宽 | 单条 PUBLISH 达 MB 级，阻塞其他消息 | 载荷用 CBOR/protobuf 压缩，或传对象存储引用 URI |
| 标志位不符 | CONNECT 后立刻被服务端断开 | 检查 PUBREL/SUBSCRIBE/UNSUBSCRIBE 的 `flags=0010`，保留位必须为 0 |
| 口令明文 | 被抓包直接读到凭据 | 8883/TLS + 双向证书，或短期 Token（用户名/口令承载 JWT） |

---

## 🖼️ 9. MQTT 报文结构与 QoS 时序图

![[MQTT.excalidraw]]

---

## 🔗 思考与延伸索引

- [[TCP]] —— MQTT 的可靠字节流底座：粘包/半包（MQTT 靠剩余长度定界）、Nagle 与心跳时延、TIME_WAIT 与重连风暴的关系
- [[HTTP]] —— 对照模型：请求/响应 vs 发布/订阅；MQTT over WebSocket 如何借用 HTTP 完成升级握手
- [[UDP]] —— 反向对照：MQTT 不使用 UDP；需 UDP 语义时看 MQTT-SN / CoAP 一类面向数据报的协议
- [[TLS]] —— 8883 端口下的证书与 SNI，用户名/口令为何必须加密传输
- [[Modbus]] —— 同一台设备上"现场总线采集 + MQTT 上云"的两段式网关架构

---

## 📚 参考资料

1. **OASIS Standard** — *MQTT Version 3.1.1*（2014-10-29，Part 1: Message Transport / Part 2: Security / Part 3: Compliance）。剩余长度编码、Connect Flags、14 类报文与标志位硬性要求的规范原文。
2. **OASIS Standard** — *MQTT Version 5.0*（2019-03-06）。Reason Code、Properties、共享订阅 `$share`、Clean Start 与 Session Expiry Interval。
3. **IBM Developer Works** — *MQTT Version 3.1.1 Part 1*（v3.1 原始规范与历史沿革）。
4. **Samuel Reed** — *Getting Started with MQTT*（O'Reilly Media，2022，免费电子书）。主题设计与 Broker 生态实践。
5. **The MQTT Bros / HiveMQ** — *MQTT Essentials*（Liam Clegg 等）与 *MQTT Tutorial* 系列：QoS 状态机与交互图例。
6. **Ralph Bakker、JP (Hivemq) 等，另见中文译介《图解物联网》（人民邮电出版社）** —— 中文《MQTT 实战》类入门读物版本繁多、术语译名不统一，涉及字段位域与状态机时**一律以 OASIS 规范原文为准**。
7. **Eclipse Mosquitto Documentation** / **EMQX Docs** — 命令行、配置项（`max_queued_messages`、persistence、ACL）与 `$SYS` 主题清单。
