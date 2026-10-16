---
title: Socket 网络编程完整技术手册（BSD Sockets API）
tags:
  - Network
  - Socket
  - POSIX
  - TCP
  - UDP
  - epoll
status: Completed
---

# 🔌 Socket 网络编程完整技术手册（BSD Sockets API）

> [!INFO] 笔记概览
> - **定位**：Socket 不是一种协议，而是操作系统内核网络栈对用户态暴露的**编程接口抽象**（BSD Sockets API，POSIX.1g 标准化）。它把 TCP/IP 协议栈的复杂行为收敛为一组文件描述符（fd）上的系统调用。
> - **内容范围**：接口本质 · 地址与字节序 · TCP/UDP 调用流与内核状态机对应 · 返回值与错误语义 · 阻塞模型与 IO 多路复用（select/poll/epoll） · 套接字选项 · 半关闭与粘包 · 嵌入式 lwIP 对照
> - **推荐路径**：§1 接口本质 → §2 地址字节序 → §3 TCP 全流程 → §4 UDP → §5 错误处理 → §6 多路复用 → §7 选项 → §8 常见坑

---

## 🧩 1. Socket 是什么：接口本质

### 1.1 三层视角

| 层 | 角色 |
| :--- | :--- |
| 应用程序 | 通过 `socket()/bind()/connect()` 等系统调用读写**字节流/数据报** |
| 内核 | Socket 层维护 fd ↔ 协议控制块（PCB/TCP 控制块），向下驱动协议栈 |
| 驱动/网卡 | 收发帧，DMA 搬运 |

- **一切皆 fd**：`socket()` 返回一个非负整数文件描述符，后续 `read/write/close` 与操作普通文件共用同一套 VFS 接口——这与 mini_tree 中设备统一走 VFS 句柄的设计同源，见 [[DMA、Buffer 与流量控制]]。
- Socket 是**端点（endpoint）**：由协议族 + 类型 + 协议号三要素确定，五元组（协议、源 IP、源端口、目的 IP、目的端口）唯一标识一条连接。

### 1.2 socket() 三参数

```c
int fd = socket(int domain, int type, int protocol);
```

| 参数 | 常用取值 | 含义 |
| :--- | :--- | :--- |
| **domain** | `AF_INET` / `AF_INET6` / `AF_UNIX` | 地址族：IPv4 / IPv6 / 本机进程间 |
| **type** | `SOCK_STREAM` / `SOCK_DGRAM` / `SOCK_RAW` | 字节流(TCP) / 数据报(UDP) / 原始套接字 |
| **protocol** | 通常填 `0` | 由 domain+type 自动选定；`SOCK_RAW` 时可指定 IPPROTO_ICMP 等 |

**type 语义对比**：

| 组合 | 语义 | 对应协议 | 典型用途 |
| :--- | :--- | :--- | :--- |
| `AF_INET + SOCK_STREAM` | 面向连接、可靠、有序字节流 | TCP | HTTP、SSH |
| `AF_INET + SOCK_DGRAM` | 无连接、不可靠数据报、保留边界 | UDP | DNS、DHCP |
| `AF_INET + SOCK_RAW` | 绕过传输层直收 IP 包 | — | ping、traceroute、抓包 |
| `AF_UNIX + SOCK_STREAM` | 本机管道式通信，无网卡参与 | — | 进程间 IPC |

---

## 📍 2. 地址体系与字节序

### 2.1 sockaddr 通用结构与 IPv4 具体结构

```c
struct sockaddr {            /* 通用形参，仅做类型转换 */
    sa_family_t sa_family;   /* AF_INET 等 */
    char        sa_data[14];
};

struct sockaddr_in {         /* IPv4 专用，成员布局与上面首字节兼容 */
    sa_family_t sin_family;  /* 恒为 AF_INET */
    in_port_t   sin_port;    /* 端口，16 bit，网络字节序 */
    struct in_addr sin_addr; /* 32 bit IP，网络字节序 */
    char        sin_zero[8]; /* 填充对齐，通常 memset 置 0 */
};
```

所有需要传地址的系统调用（`bind/connect/accept/sendto`）形参都是 `struct sockaddr *`，用 `(struct sockaddr *)&addr_in` 强制转换——这是 C 手写的"多态"，真正的结构体由协议族决定。

### 2.2 字节序转换（网络大端序）

| 函数 | 用途 |
| :--- | :--- |
| `htons / htonl` | 主机序 → 网络序（16/32 位）：端口、IP |
| `ntohs / ntohl` | 网络序 → 主机序 |
| `inet_pton / inet_ntop` | 点分十进制字符串 ↔ 二进制地址（IPv4/IPv6 通用，取代废弃的 `inet_addr`） |

> [!WARNING] 忘记 htons 是入门第一大坑
> x86/ARM 默认小端。`sin_port = 80` 不经 `htons` 会变成端口 20480（0x0050 → 0x5000）。抓包时见到"80 端口连不上、20480 有 SYN"即此症状。

---

## 🔁 3. TCP 调用流与内核状态机对应

### 3.1 服务端流程（被动打开）

```text
服务端                          客户端
socket()                        socket()
setsockopt(SO_REUSEADDR)          |
bind()                            |
listen()   ──── 内核进入 LISTEN    |
accept() 阻塞等待                 connect() ──> SYN
    <───── 三次握手完成 ────────────  
accept() 返回新 fd               ESTABLISHED
read()/write() 收发字节流        read()/write()
close()      ──── FIN 交换 ────  close()
```

一个关键设计：**`accept()` 返回的是"已连接套接字"新 fd**，监听 fd 继续接下一连接——所以单线程服务器 `listen_fd` 永不读写数据。

### 3.2 服务端最小实现

```c
int srv = socket(AF_INET, SOCK_STREAM, 0);
int on = 1;
setsockopt(srv, SOL_SOCKET, SO_REUSEADDR, &on, sizeof on); /* 重启秒bind */

struct sockaddr_in addr = {0};
addr.sin_family = AF_INET;
addr.sin_port   = htons(8080);
addr.sin_addr.s_addr = htonl(INADDR_ANY);   /* 监听所有网卡 */

bind(srv, (struct sockaddr *)&addr, sizeof addr);
listen(srv, 128);                            /* backlog：全连接队列上限 */

for (;;) {
    struct sockaddr_in peer;
    socklen_t len = sizeof peer;
    int cli = accept(srv, (struct sockaddr *)&peer, &len); /* 阻塞 */
    handle(cli);
    close(cli);
}
```

### 3.3 客户端流程

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in srv = {0};
srv.sin_family = AF_INET;
srv.sin_port   = htons(80);
inet_pton(AF_INET, "93.184.216.34", &srv.sin_addr);

connect(fd, (struct sockaddr *)&srv, sizeof srv); /* 阻塞至三次握手完成 */
write(fd, buf, n);
read(fd, buf, sizeof buf);
close(fd);
```

### 3.4 API 调用 ↔ TCP 状态对照

| 调用/事件 | 本端状态变化 | 对端可见报文 |
| :--- | :--- | :--- |
| `socket()` | CLOSED | — |
| `bind()+listen()` | → LISTEN | — |
| `connect()` 发起 | → SYN_SENT | SYN |
| 收到 SYN+ACK | → SYN_RECEIVED | ACK |
| `accept()` 返回 | → ESTABLISHED | ← 对端 ACK |
| `close()`（无未读数据） | → FIN_WAIT_1 | FIN |
| 收到 FIN 并回 ACK 后主动关 | LAST_ACK → CLOSED | FIN |
| 被动方收完再关 | TIME_WAIT（等 2MSL）→ CLOSED | 最后 ACK |

细节与状态机全图见 [[TCP]]。

---

## 💬 4. UDP 调用流

```text
服务端                          客户端
socket()                        socket()
bind(port 53)                   （可不 bind，connect 也可）
recvfrom() 阻塞等待             sendto("8.8.8.8:53")
    <─── 数据报到达，保留边界 ───
sendto(回包，带目的地址)        recvfrom()
close()                         close()
```

- 无 `listen/accept/connect`（UDP 的 `connect()` 只是**固定对端**的内核优化：之后可用 `read/write`，并过滤四元组——常用于 DNS 客户端只收自己发起的应答，见 [[UDP]]）。
- 每次 `sendto/recvfrom` 都携带目的/源地址；一个 fd 可同时服务任意多对端——这是 UDP 服务器不需要 accept 新 fd 的原因。

---

## 🚨 5. 返回值与错误语义

### 5.1 通用约定

- 成功：返回 **≥0**（字节数或 fd）；失败：返回 **-1**，具体原因看 `errno`。
- `read/recv` 返回 **0 = 对端正常关闭**（EOF）——不检查这一条，循环读会死循环。
- 返回 **-1 且 errno == EINTR**：被信号打断，应当重试。

### 5.2 高频 errno 表

| errno | 含义 | 触发场景与处理 |
| :--- | :--- | :--- |
| `EAGAIN / EWOULDBLOCK` | 非阻塞下暂无数据/发送缓冲满 | **不是错误**：回到 epoll/select 等可读写 |
| `EINTR` | 调用被信号打断 | 重试本调用 |
| `EPIPE` | 对端已关闭还写 | 默认进程直接收到 SIGPIPE 被杀！写大响应必须 `MSG_NOSIGNAL` 或屏蔽信号 |
| `ECONNRESET` | 对端 RST | 丢弃本连接，清理状态 |
| `EADDRINUSE` | 端口被占 | bind 前设 `SO_REUSEADDR`；确认没有僵尸监听进程 |
| `EACCES` | 权限不足 | 绑定 <1024 特权端口需 root/CAP_NET_BIND_SERVICE |
| `ETIMEDOUT` | connect 超时 | 目标不可达/被防火墙静默丢包 |

### 5.3 短读短写：TCP 不保证一次搬完

`write(fd, buf, 1000)` 可能只写了 600 字节就返回；`read()` 返回的是"当前内核缓冲里有多少给多少"。**发送/接收整个缓冲区必须循环**：

```c
/* 可靠写整块（非阻塞 fd 需配合 epoll 等待可写） */
ssize_t write_all(int fd, const char *buf, size_t n)
{
    size_t off = 0;
    while (off < n) {
        ssize_t k = write(fd, buf + off, n - off);
        if (k > 0) { off += k; continue; }
        if (k < 0 && errno == EINTR) continue;
        if (k < 0 && (errno == EAGAIN || errno == EWOULDBLOCK)) return off; /* 交回事件循环 */
        return -1;
    }
    return (ssize_t)off;
}
```

### 5.4 粘包 / 拆包是应用层问题

TCP 是字节流，没有消息边界；"粘包"并非 TCP 的 bug，而是**消息分帧本来就是应用层协议的责任**。三种标准分帧法：

| 方法 | 代表协议 | 要点 |
| :--- | :--- | :--- |
| 固定长度 | 部分二进制协议 | 简单，浪费空间 |
| 分隔符 | HTTP 头部 `\r\n\r\n`、文本行 `\n` | 数据内容不能含分隔符，需转义 |
| **长度前缀** | MQTT 剩余长度、Protobuf、RPC | 最通用：头里放 payload 字节数 |

---

## ⚙️ 6. 阻塞模型与 IO 多路复用

### 6.1 四种 IO 模型

| 模型 | 结构 | 缺点 |
| :--- | :--- | :--- |
| 阻塞 BIO + 每连接一线程/进程 | 代码最直白（如串口调试工具） | fd 多了线程栈爆炸；C10K 不成立 |
| 非阻塞 NBI + 轮询 | `O_NONBLOCK` + 死循环试 | CPU 空转 |
| IO 多路复用 | `select/poll/epoll` 一次等一组 fd | 单线程逻辑要状态机化 |
| 信号驱动 AIO / io_uring | 内核就绪时通知 | 复杂度最高 |

### 6.2 select / poll / epoll 对比

| 维度 | select | poll | epoll(LT/ET) |
| :--- | :--- | :--- | :--- |
| fd 上限 | 1024（`FD_SETSIZE`） | 无（链表） | 无 |
| 就绪判定复杂度 | O(n) 每次全量拷贝集合 | O(n) | **O(1)** 回调投递就绪链表 |
| 内核↔用户数据搬运 | 每调用全量拷贝位图 | 全量拷贝数组 | `CTL_ADD/MOD/DEL` 注册一次 |
| 边缘触发 ET | ✗ | ✗ | ✓（必须配非阻塞 + 读到 EAGAIN） |
| 适用 | 跨平台小并发 | 少量 fd | Linux 高并发服务器主力 |

### 6.3 epoll 事件循环骨架

```c
int ep = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN | EPOLLET, .data.fd = listen_fd };
epoll_ctl(ep, EPOLL_CTL_ADD, listen_fd, &ev);

struct epoll_event events[64];
for (;;) {
    int n = epoll_wait(ep, events, 64, timeout_ms); /* -1 永久等 */
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listen_fd) {
            /* accept 新连接，非阻塞循环 accept 到 EAGAIN */
        } else {
            /* 处理读写；ET 模式必须一次性 read/write 到 EAGAIN */
        }
    }
}
```

> [!NOTE] 模型选择经验
> 连接少、逻辑重（协议解析器）→ 线程池 + 阻塞 IO 最省心；连接多、逻辑轻（代理、消息服务器）→ epoll 单/多 Reactor。MQTT broker（[[mqtt]]）、HTTP 服务器（[[http]]）都是后者。

---

## 🎛️ 7. 常用套接字选项速查

`setsockopt(fd, level, optname, &val, len)`：

| 选项 | level | 作用 | 建议 |
| :--- | :--- | :--- | :--- |
| `SO_REUSEADDR` | SOL_SOCKET | TIME_WAIT 端口可立即重 bind；允许多播复用 | **服务端必设**，否则重启即 `EADDRINUSE` |
| `SO_LINGER` | SOL_SOCKET | 改 close 行为：linger=0 立即发 RST 丢未读数据 | 调试用，生产慎用 RST |
| `SO_KEEPALIVE` | SOL_SOCKET | 打开 TCP 保活探测 | 中间 NAT 会静默删会话，长连接必开 |
| `TCP_KEEPIDLE/INTVL/CNT` | IPPROTO_TCP | 保活参数：默认 idle 7200 s 太长，常调 60 s 起 | 与应用层心跳（MQTT Keep Alive）二选一即可 |
| `TCP_NODELAY` | IPPROTO_TCP | 关闭 Nagle，小包立发 | 低时延交互协议（RPC、游戏）必开；批量传输别开 |
| `SO_RCVBUF / SO_SNDBUF` | SOL_SOCKET | 收发缓冲大小，影响带宽时延积 | 高吞吐链路配合窗口缩放 |
| `TCP_CORK / TCP_QUICKACK` | IPPROTO_TCP | 攒满 MTU 再发 / 取消延迟确认 | 与 [[TCP]] 中 Nagle×延迟确认交互对照 |
| `MSG_PEEK / MSG_DONTWAIT`（recv flag） | — | 窥探不移走数据 / 单次非阻塞 | 先探协议头再决定读多少 |

---

## 🪤 8. 半关闭、TIME_WAIT 与常见坑清单

### 8.1 close vs shutdown

- `close(fd)`：引用计数 -1；若该 fd 还有未读数据，可能直接发 **RST** 让对端丢数据。
- `shutdown(fd, SHUT_WR)`：**只关写方向**，FIN 照发，读方向仍可收对端剩余数据——"请求发完了还等完整响应"的标准姿势（HTTP 短请求、写读半双工协议）。
- socket fd 被 `dup` 或多线程共享时，确认唯一所有者再 close，否则出现"提前关"竞态。

### 8.2 服务端 TIME_WAIT 处理

- TIME_WAIT 在**主动关闭方**，服务端大量主动关连接 → 端口/句柄吃紧。策略：
  1. 先问"为什么服务端主动关"——多数应由客户端关或复用连接（HTTP keep-alive）；
  2. 开 `SO_REUSEADDR` 解 bind 阻塞；
  3. 内核参数 `tcp_tw_reuse`（出方向）辅助；`tcp_tw_recycle` IPv4+NAT 下丢包，**已删除，别再用**。

### 8.3 高频事故对照表

| 症状 | 大概率根因 |
| :--- | :--- |
| 服务端重启 bind 失败 | 没设 `SO_REUSEADDR` |
| 写大响应偶发进程整个退出 | SIGPIPE 未屏蔽/未用 `MSG_NOSIGNAL` |
| 请求响应慢 ~40 ms | Nagle × 对端延迟确认交互，未设 `TCP_NODELAY` |
| 客户端断线但服务端 recv 永不返回 | 对端掉电没发 FIN，必须靠心跳/保活发现 |
| 长连接"半夜"静默断开 | NAT 会话老化，需应用层心跳 < NAT 超时 |
| epoll ET 模式只处理第一个事件 | 未循环读到 `EAGAIN`；ET 的标配是非阻塞+读尽 |
| accept 风暴（100% CPU） | 边缘触发漏 `EPOLLOUT`，或 accept 未到 EAGAIN |

---

## 🧊 9. 嵌入式对照（一句话）

- **lwIP** 提供 `api.h`（netconn 抽象）与 `sockets/` BSD socket 兼容层，语义与本手册一致；裸机侧注意 `LWIP_SO_RCVBUF`、sys_arch 邮箱队列与本仓 mini_tree VFS 句柄模型的差别——MCU 上 socket 数量与缓冲是池化定容的，对应 [[DMA、Buffer 与流量控制]]。

---

## 🖼️ 10. Socket 调用流与 TCP 生命周期时序图

![[Socket.excalidraw]]

---

## 🔗 思考与延伸索引

- [[TCP]] — connect/close 背后真正的状态机、重传与拥塞控制
- [[UDP]] — sendto/recvfrom 的无连接语义与 ICMP 端口不可达
- [[IP]] — 内核路由表查询与分片，socket 之下发生了什么
- [[http]] — 长度前缀/分块传输分帧、keep-alive 复用实例
- [[mqtt]] — 应用层 Keep Alive 心跳与 TCP keepalive 的取舍
- [[SSH]] — 一条 TCP 上多 channel 多路复用的上层范本
- [[DHCP]] — UDP 广播交互：端口 bind 与广播标志位实战
- [[DMA、Buffer 与流量控制]] — 缓冲区满/溢出的通用模型（socket 缓冲同源）
- [[SPI]] — 同步外设无"字节流边界"概念，对照 TCP 流的抽象差异

---

## 📚 参考资料

1. **W. Richard Stevens, Bill Fenner, Andrew Rudoff** — *UNIX Network Programming, Volume 1: The Sockets Networking API (3rd Ed.)*（第 1~6、14~16 章：API、错误、ioctl、多路复用）.
2. **Michael Kerrisk** — *The Linux Programming Interface*, Ch.57~60（epoll、socket 选项）.
3. **Linux man-pages** — `man 7 socket`、`man 7 tcp`、`man 7 udp`、`man 7 epoll`、`man 2 connect`（EINPROGRESS 语义）.
4. **POSIX.1-2017 (IEEE Std 1003.1)** — BASE Interfaces, Sockets（`<sys/socket.h>` 规范来源）.
5. **IETF** — RFC 9293（TCP 状态机与半关闭）；RFC 6335（端口注册策略，对应 bind 选择）.
6. **D. J. Bernstein** — *Beej's Guide to Network Programming*（入门向快速参考）.
