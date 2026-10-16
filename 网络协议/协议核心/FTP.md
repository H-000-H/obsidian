---
title: FTP 协议完整技术手册
tags:
  - Network
  - Protocol
  - FTP
  - Application-Layer
  - File-Transfer
  - PlainText
status: Completed
---

# 🔗 FTP 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：FTP（File Transfer Protocol，文件传输协议）是运行在 **TCP 之上**的应用层协议，最大特征是使用**两条独立 TCP 连接**——**控制连接（Control Connection，默认 21 端口，贯穿整个会话）**与**数据连接（Data Connection，每次传输临时建立、传完即关）**；控制信道走 CRLF 结尾的 NVT-ASCII 文本命令，数据信道只放纯文件字节。
> - **现状判断**：FTP 是 **1985 年 RFC 959 定型的明文协议**，用户名/口令/文件内容全部可被抓包读取，现代环境由 **FTPS（FTP over TLS，RFC 4217）** 与 **SFTP（SSH File Transfer Protocol，承载于 [[SSH]]）** 替代；但因其双通道语义简单、服务端存量巨大、内网设备/老系统只支持裸 FTP，仍必须掌握。
> - **内容范围**：双通道模型 · 行协议与 TYPE 模式 · 命令全集 · 应答码体系 · 主动/被动与 EPRT/EPSV · 完整会话时间线 · REST 断点续传与匿名 FTP · 被动端口范围与安全加固
> - **推荐路径**：§1 定位 → §2 交互模型 → §3 命令表 → §4 应答码 → §5 主动 vs 被动（重点） → §6 会话时间线 → §7 续传与部署 → §8 安全 → §9 图

---

## 🧭 1. 协议定位与双通道设计

### 1.1 核心属性

| 属性 | 说明 |
| :--- | :--- |
| 定义 | FTP（File Transfer Protocol），规范主线：RFC 114（1971）→ RFC 765 → **RFC 959**（1985，现行） |
| 所在层 | 应用层 / Application Layer（OSI 第 7 层），依赖 [[TCP]] 提供可靠字节流 |
| 服务模型 | Client–Server（客户机/服务器），**有状态会话**（服务端保存登录态、当前目录、TYPE/STRU/MODE 参数） |
| 端口 | **21/TCP = `ftp`（控制连接）**；**20/TCP = `ftp-data`（主动模式数据连接的服务器源端口，约定值）** |
| 连接数 | **≥ 2 条 TCP 连接**：1 条控制 + 每条文件传输各 1 条数据（不复用、传完关闭） |
| 命令语法 | 3~4 个大写字母助记符 + 空格 + 参数，**以 CRLF 结尾的一行一条命令**（Telnet NVT 约定）；应答为 3 位码 + 文本 |
| 数据表示 | 由 `TYPE`/`STRU`/`MODE` 三参数决定，实践中 **TYPE I + MODE S（流模式）** 占绝大多数 |
| 寻址能力 | 命令里直接嵌 IP 与端口（`PORT`/`PASV` 参数），这是 NAT/防火墙时代所有麻烦的根源 |
| 承载上层 | 站点发布、固件/日志批量落盘、老旧 M2M 设备回传、公开镜像站（`ftp://` scheme） |

### 1.2 控制连接与数据连接：为什么拆成两条

```text
              控制连接（21/TCP，全程 1 条，命令/应答）
  ┌──────────────────────────────────────────────────────────┐
  │  USER/PASS  PWD  CWD  LIST  TYPE  PASV  RETR  QUIT  ...   │
  └──────────────────────────────────────────────────────────┘
        │            │                 │
        │ 每次传输临时开一条数据连接（传完立即 FIN 关闭）
        ▼            ▼                 ▼
   数据连接 #1    数据连接 #2       数据连接 #3
   (LIST 目录)    (RETR a.bin)      (STOR b.log)
```

| 维度 | 控制连接（Control Connection） | 数据连接（Data Connection） |
| :--- | :--- | :--- |
| 生命周期 | **整个会话保持**，从 TCP 建链到 `QUIT` | **一次传输一条**，EOF/关闭即结束 |
| 端口 | 服务器 **21**（可配） | 服务器 **20**（主动）或**随机高端口**（被动） |
| 内容 | 明文命令 + 明文应答（含**用户名与口令**） | **只有文件字节**，无协议头、无分隔符 |
| 结束判定 | 由 `QUIT`/超时/`421` 决定 | **靠 TCP 连接关闭（EOF）** 表示传输结束 |
| 半关闭语义 | 全双工 | 上传时客户端常 `shutdown(SHUT_WR)` 半关闭通知服务端"发完了" |
| 失败码 | `421`、`530` | `425`（连不上）、`426`（中途断） |

> [!IMPORTANT] 「数据结束 = 连接关闭」是 FTP 的设计命门
> 数据信道上没有任何长度字段或结束标记，接收方只能靠 **TCP FIN** 判断文件传完。这一方面让 FTP 无法在一条数据连接上连续传多个文件（HTTP 的持久连接做不到这一点，但 FTP 也为此付出了每文件一次三次握手的代价）；另一方面，**中间设备提前拆链会造成"静默截断"**：客户端只看到 226/250，文件其实不完整——所以生产环境永远要用 `SIZE`/`MDTM` 或校验和复核。

### 1.3 明文协议的现状：FTPS / SFTP 替代趋势

| 方案 | 承载与端口 | 加密范围 | 与 RFC 959 的关系 | 现实评价 |
| :--- | :--- | :--- | :--- | :--- |
| **裸 FTP** | TCP 21 + 20/随机 | **无（全明文）** | 本体 | 仅剩内网/设备兼容场景；公网应视为禁用 |
| **FTPS 显式（Explicit）** | 控制 **21**、数据随机 | 命令+数据均可 TLS | `AUTH TLS` 在 959 之上升级同一连接，兼容老端口 | RFC 4217 + RFC 2228；**仍被 NAT/防火墙双通道折磨** |
| **FTPS 隐式（Implicit）** | 控制 **990**、数据 **989** | 建链即 TLS，无明文握手 | 非标准（历史草案），无 `AUTH` 协商 | 逐步淘汰，客户端需显式指定 `ftps://`/`ftpes://` |
| **SFTP** | **SSH 22 单端口** | 全量加密 + 完整性 | **完全不同的协议**（draft-ietf-secsh-filexfer），不是"FTP over SSH" | 事实上的现代标准：单通道、天然穿 NAT、可密钥认证 |
| **SSH 端口转发承载 FTP** | 本地 21 → 远端 21 | 隧道内加密 | 不改协议 | 可行但笨重，通常直接换 SFTP |

> [!WARNING] 术语陷阱：FTP over SSH ≠ SFTP
> 有人用"FTP over SSH"描述"用 [[SSH]] 隧道转发 21 端口的裸 FTP"；而 **SFTP 是 SSH 的文件传输子系统**，只有**一条 TCP 连接**、二进制帧、没有 PORT/PASV 概念。排障时先确认到底是哪一种，两者的抓包特征（21 端口明文 `USER` 行 vs 22 端口 `SSH-2.0-` 横幅）一眼可辨。

---

## 📜 2. 交互模型：CRLF 行协议与 TYPE 模式

### 2.1 行协议语法

| 要素 | 规则 | 示例 |
| :--- | :--- | :--- |
| 命令长度 | **一行一命令**，必须以 `<CRLF>`（`0x0D 0x0A`）结尾 | `RETR report.pdf\r\n` |
| 助记符 | 3~4 字母，**服务端应对大小写不敏感** | `retr` / `RETR` 等价 |
| 参数分隔 | 助记符与参数之间**恰好一个空格**，参数内部空格原样保留（文件名可含空格） | `STOR my report 2026.txt` |
| 参数内 CRLF | 除个别命令（`PASS`、账号信息）外，参数**不得含 CR/LF** | — |
| 命令与应答配对 | 每条命令**必须**得到应答；客户端一般**串行**等待应答再发下一条（流水线会错配） | — |
| 编码 | 7-bit NVT-ASCII；非 ASCII 文件名靠 `OPTS UTF8 ON`（RFC 2640）协商 | `200 UTF8 set to on` |

> [!TIP] 为什么 FTP 客户端不能并发发命令
> 控制连接没有请求/应答关联 ID，应答只按到达顺序对应。若并发 `RETR a`、`RETR b`，`150`/`226` 就无法判定属于哪个文件。FTP 因此是**严格锁步（lock-step）**协议，一个会话里的吞吐被 RTT 严重限制——这正是批量传输要靠 `lftp mirror -j N` 开 **N 条会话**（N 个控制连接）并发的原因。

### 2.2 TYPE：ASCII 模式与图片（二进制）模式

`TYPE` 设置的是**数据信道的字节解释方式**，两种参数在 99% 的场景里就是全部：

| 参数 | 全称 | 换行处理 | 位宽处理 | 适用 |
| :--- | :--- | :--- | :--- | :--- |
| **A** | ASCII / Text（网络虚拟终端 NVT-ASCII） | 传输时转成 **CRLF**；接收端转成本地惯例（Unix `LF`、Windows `CRLF`、Mac OS 9 `CR`） | 强制 **7 bit**（高位用于奇偶填充约定） | 纯文本、CSV、shell 脚本、`LIST` 输出 |
| **I** | Image（八位组/octet stream，"图片模式"） | **不改动任何字节** | 8 bit 原样 | 压缩包、镜像、图片、可执行文件、数据库备份——**默认该用它** |
| E / L | EBCDIC / Local byte size | `TYPE E N/R/L` 做字符集转换；`TYPE L n`（n=8）不做换行转换 | EBCDIC 为大型机遗留；L 按 n 位传原始字节 | 现代服务端基本返回 `504 Command parameter not implemented` |

应答：`200 Switching to Binary mode.` / `200 Type set to A.`（`TYPE` 本身**不产生数据连接**，只改会话状态）。

> [!CAUTION] 经典事故：用 ASCII 模式传二进制文件
> `TYPE A` 下服务端会把文件里每个 `0x0A` 展开为 `0x0D 0x0A`，反向亦然。**ZIP/PNG/JAR/固件镜像经 ASCII 模式传输即被静默改坏**，文件大小变化、CRC 校验失败。反之，用 `TYPE I` 传文本文件是安全的（只是不做换行转换）。因此现代客户端默认一律 `TYPE I`，只有明确需要跨平台换行转换时才用 `TYPE A`；`REST` 的续传单位也随 TYPE 变化（§7.1）。

### 2.3 STRU 与 MODE：另外两个传输参数

| 命令 | 取值 | 含义 | 现状 |
| :--- | :--- | :--- | :--- |
| `STRU` | `F` / `R` / `P` | 文件结构：无结构（File）/ 记录（Record）/ 分页（Page） | **只有 `F` 被普遍支持**（默认即 F） |
| `MODE` | `S` / `B` / `C` | 传输模式：流（Stream）/ 块（Block）/ 压缩（Compressed） | **只有 `S` 被普遍支持**；B/C 在 RFC 959 里预留，现代服务端多返回 `504 Command parameter not implemented` |

```text
C: SYST
S: 215 UNIX Type: L8            ← 仅 IANA 注册的系统名占位串，**不是文件名大小写判据**（要看服务端文件系统）
C: FEAT                          ← RFC 2389：能力协商
S: 211-Extensions supported: UTF8 TIME MDTM SIZE REST STREAM MLSD MLST EPSV EPRT AUTH TLS PBSZ PROT
S: 211 END
```

---

## 🛠️ 3. 命令全集（Command Reference）

### 3.1 认证与会话管理

| 命令 | 参数 | 用途与要点 |
| :--- | :--- | :--- |
| `USER` | `<username>` | 提交用户名。成功返回 `331`（要口令）或直接 `230`（口令走 `PASS` 可选）；可**会话中重发**切换用户（RFC 959 允许 `USER` 中断前次登录） |
| `PASS` | `<password>` | 提交口令，**明文**；`USER alice` → `331` → `PASS s3cr3t` → `230 Login successful.`；口令错误 `530 Not logged in` |
| `ACCT` | `<account>` | 补充账号/项目信息（仅当服务端返回 `332 Need account for login` 或 `532 Need account for storing files` 时才需要） |
| `CWD` | `<path>` | Change Working Directory，切到**相对或绝对**目录；成功 `250` |
| `CDUP` | 无 | 等价 `CWD ..`，返回上级；成功 `250 Directory successfully changed.` |
| `SMNT` / `REIN` | `<pathname>` / 无 | SMNT：Structure Mount（记录型文件系统遗留），现代多回 `502`；REIN：Reinitialize，注销用户并清空会话参数但**保持控制连接**，成功回 `220` |
| `QUIT` | 无 | 结束会话，服务端回 `221 Goodbye.` 后关闭控制连接 |
| `NOOP` | 无 | 空操作，回 `200`；用于保活（防 `idle_session_timeout`）与 RTT 探测 |
| `SYST` / `HELP` | 无 / `[<cmd>]` | SYST 查服务端系统类型（回 `215 UNIX Type: L8`）；HELP 查命令帮助（回 `211`/`214`） |
| `FEAT` | 无 | RFC 2389：列出扩展能力（`UTF8`/`SIZE`/`MDTM`/`MLSD`/`EPSV`/`AUTH TLS`…），回 `211-…-211 END` |
| `OPTS` | `<cmd> <params>` | RFC 2389：打开某扩展选项，如 `OPTS UTF8 ON` → `200`；`OPTS MLST type;size;modify;` |
| `STAT` / `SITE` | `[<dataset>]` / `<subcmd>` | STAT 查**当前会话/传输状态**（不建数据连接，回 `211`/`212`/`213`，带参数时类似 `LIST`）；SITE 为厂商自定义子命令（`SITE CHMOD 644 f`、`SITE IDLE 300`），非标准 |

### 3.2 目录浏览与文件元信息

| 命令 | 参数 | 用途与要点 |
| :--- | :--- | :--- |
| `PWD` | 无 | 打印当前目录，回 `257 "/" is the current directory`（RFC 原文语义是 `<257> "PATHNAME" created`，实现普遍借用为 PWD 应答） |
| `LIST` | `[<path>]` | **给人看**的目录列表：权限/链接数/属主/大小/日期/名字。**需建数据连接**，内容随服务端与 `ls -l` 格式而变（不可稳定解析） |
| `NLST` | `[<path>]` | **只给名字**（一行一个，CRLF 结尾），比 `LIST` 易解析，但无大小/时间。**需建数据连接** |
| `MLSD` / `MLST` | `[<path>]` / `<path>` | RFC 3659：**机器可读**列表与单条目查询；MLSD 每行 `type=dir;modify=20260921081530; name/` 形式可解析（需数据连接，现代客户端优先），MLST 回 `250- type=...` + `250 End` |
| `SIZE` | `<filename>` | RFC 3659：查文件字节数，回 `250 482190`；断点续传与完整性复核的关键 |
| `MDTM` | `<filename>` | RFC 3659：查最后修改时间（UTC，`YYYYMMDDHHMMSS`），回 `213 20260921081530`；判断"是否需要重传" |
| `RMD` | `<path>` | Remove Directory（**只能删空目录**），回 `250`；失败 `550 Directory not empty` |
| `MKD` | `<path>` | Make Directory，回 `257 "/new/dir" created` |

### 3.3 文件传输与改名/删除

| 命令 | 参数 | 用途与要点 |
| :--- | :--- | :--- |
| `RETR` | `<filename>` | **Retrieve（下载）**：开数据连接，服务端把文件字节推给客户端，结束回 `226` |
| `STOR` | `<filename>` | **Store（上传）**：客户端往数据连接写；**目标存在则覆盖/截断**，结束回 `226` |
| `APPE` | `<filename>` | Append：追加写入（日志续传、上传分段拼接），语义与 `STOR` 同但不清空 |
| `STOU` | 无参 | Store Unique：服务端生成唯一文件名上传，回 `250 FILE: <生成的名字>`（老式"投递箱"用法）；`ALLO`（预留空间）几乎无实现，回 `202` 可忽略 |
| `RNFR` | `<from>` | **Rename From**：改名第一步，回 `350 Requested file action pending further information.` |
| `RNTO` | `<to>` | **Rename To**：紧接 `RNFR` 完成改名，回 `250`；两者是**成对的两步命令**（FTP 没有单条 rename） |
| `DELE` | `<filename>` | 删文件（不删目录，目录用 `RMD`），回 `250` |
| `ABOR` | 无 | 中止当前传输：数据连接被关闭，常见应答序列 `426 Connection closed; transfer aborted` + `226/225`；控制连接仍可用 |
| `REST` | `<mark>` | **断点续传定位**：设置重启标记，**必须紧随其后一条 `RETR`/`STOR`/`APPE`** 才生效（§7.1） |

### 3.4 数据连接建立（四种）

| 命令 | 参数 | 用途与要点 |
| :--- | :--- | :--- |
| `PORT` | `h1,h2,h3,h4,p1,p2` | **主动模式**：客户端告知自己的 IP 与端口，让**服务端反向连过来**；端口 $=p_1\times256+p_2$；回 `200 PORT command successful` |
| `PASV` | 无 | **被动模式**：请求服务端开一个监听端口，回 `227 Entering Passive Mode (h1,h2,h3,h4,p1,p2)`，由**客户端连过去** |
| `EPRT` | `\|<n>│<addr>│<port>\|` | RFC 2428 主动模式的**可扩展**写法，`n=1` IPv4、`n=2` IPv6，如 `EPRT \|2\|2001:db8::1\|2121\|` → `200 EPRT command successful` |
| `EPSV` | `[<n>]` | RFC 2428 被动模式可扩展写法，回 `229 Entering Extended Passive Mode (\|\|\|50000\|)`；单/双栈与 v4/v6 转换都更干净；定界符由客户端选（取命令首字符），非硬编码逗号 |

---

## 🚦 4. 应答码体系（Reply Codes）

### 4.1 五位类别总览

应答格式：`xyz<SP>文本<CRLF>`，`x` 为类别、`y` 为文档、`z` 为细节。

| 类别 | 名称 | 语义 | 典型动作 |
| :--- | :--- | :--- | :--- |
| **1yz** | Positive Preliminary（初步肯定） | 请求正在处理，**马上会开数据连接**；不应期待更多同类应答 | 等 `2yz` 收尾 |
| **2yz** | Positive Completion（完成肯定） | 请求**已成功完成** | 继续下一步 |
| **3yz** | Positive Intermediate（中间肯定） | 命令接受，但**还需要另一条命令**才完成 | 立即补 `PASS`/`RNTO` |
| **4yz** | Transient Negative Completion（临时否定） | 本次失败，**重试可能成功**（资源忙、瞬时故障） | 退避重试 |
| **5yz** | Permanent Negative Completion（永久否定） | 请求**被拒绝**，重试无意义 | 报错、停止 |

`y` 位语义（RFC 959）：`1` 语法与信息、`2` 信息与状态、`3` 会话、`4` **未使用**、`5` 系统；文件相关动作用 `x5z` 家族（450/550 等）。

### 4.2 典型码逐个说明（高频必背）

| 码 | RFC 原文语义 | 触发场景与排障方向 |
| :--- | :--- | :--- |
| `150` | File status okay; about to open data connection. | `RETR`/`STOR`/`LIST` 后**主动模式**下常见：服务端即将反向连接；后必跟 `226` |
| `125` | Data connection already open; transfer starting. | **被动模式**常见替代 `150`：数据连接已由客户端开好（`120 Service ready in nnn minutes.` 为过载排队预留，实现罕见） |
| `200` | Command okay. | `TYPE`/`STRU`/`MODE`/`PORT`/`OPTS` 等参数类命令成功 |
| `202` | Command not implemented, superfluous at this site. | `ALLO` 等；**不是错误**，客户端可忽略继续 |
| `211` | System status, or system help reply. | `FEAT`/`STAT`/`HELP` 多行应答起始（`211-` … `211 ` 结束） |
| `212` / `213` | Directory status / File status. | `STAT <dir>` / `STAT <file>`、`MDTM` 回 `213 <时间>` |
| `214` / `215` | Help message. / UNIX Type: L8 | `HELP` 输出 / `SYST` 系统标识 |
| `220` | Service ready for new user. | **控制连接建立后服务端的第一条主动欢迎**；FTPS 中也是 `AUTH TLS` 成功应答 |
| `221` / `225` | Service closing control connection. / Data connection open; no transfer in progress. | `QUIT` 成功与服务端优雅下线 / `STAT` 报告数据连接仍在、`ABOR` 之后 |
| `226` | Closing data connection. Requested file action successful. | **传输完成的标志**：数据连接已由服务端关闭 |
| `227` / `229` | Entering Passive Mode. / Entering Extended Passive Mode. | `PASV` 成功（括号内是 IP + 端口六元组）/ `EPSV` 成功（RFC 2428），形如 `(\|\|\|50000\|)` |
| `230` | User logged in, proceed. | 认证成功，会话进入 **LOGIN 态**，可发文件类命令 |
| `250` | Requested file action okay, completed. | `CWD`/`DELE`/`RNTO`/`SIZE`/`STOR`(完成时也有用 226) 等 |
| `257` | "PATHNAME" created. | `MKD` 建目录 / `PWD` 报当前目录 |
| `331` / `332` | User name okay, need password. / Need account for login. | `USER` 后**必须**紧接 `PASS`（口令不得写在 `USER` 里）/ 还需 `ACCT` |
| `350` | Requested file action pending further information. | `RNFR` 成功（等 `RNTO`）、`REST` 已接受（等传输命令） |
| `421` | Service not available, closing control connection. | 过载/超时/访问控制拒绝；**连接即将关闭**，可稍后重连 |
| `425` | Can't open data connection. | **双通道协议头号故障码**：服务端连不上 `PORT` 给的地址（主动）或客户端连不上 `227` 给的地址（被动），几乎必为防火墙/NAT/端口范围问题 |
| `426` | Connection closed; transfer aborted. | 传输中途数据连接断开（含 `ABOR`）：**文件不完整**，需 `REST` 续传；`430`（invalid nickname）为邮件转发遗留 |
| `450` / `451` | File unavailable (e.g., file busy). / Requested action aborted: local error in processing. | 暂时性：文件被占用 / 服务端内部错误（IO、限流） |
| `452` | Requested action not taken. Insufficient storage space in system. | **系统级**磁盘/inode 满或配额不足（用户级见 `552`） |
| `530` / `532` | Not logged in. / Need account for storing files. | **未登录就发文件命令**，或 `PASS` 校验失败、账号禁用 / 上传前需 `ACCT` |
| `550` / `551` | Requested action not taken. File unavailable (e.g., file not found, no access). / Page type unknown. | **头号"业务"错误码**：不存在、无权限、被 chroot 拦住、`write_enable=NO` 时 `STOR` 也被拒 / 分页模式遗留 |
| `552` / `553` | Requested file action aborted. Exceeded storage allocation. / File name not allowed. | 超用户配额 / 文件名非法或被策略过滤（`deny_file` 等） |
| `500` / `501` | Syntax error, command unrecognized. / Syntax error in parameters or arguments. | 未知命令、**行过长**、编码问题（UTF-8 未协商）/ 参数缺失或格式错（如 `PORT` 六段写五段） |
| `502` / `504` | Command not implemented. / Command parameter not implemented. | `SMNT`/`MODE B` 等整条命令不支持 / 命令认识但参数不认识（`TYPE E`、`MODE C`） |
| `503` | Bad sequence of commands. | **顺序错**：`RNTO` 没跟 `RNFR`、`REST` 后直接 `PWD`、未 `USER` 就 `PASS` |

### 4.3 多行应答的语法规则

```text
S: 211-Extensions supported:      ← NNN- 起始，表示"这是多行的第一行"
S:  EPRT
S:  IDLE
S:  MDTM
S:  SIZE
S: 211 END                        ← NNN<SP> 结束，客户端据此判定应答收完
```

| 规则 | 说明 |
| :--- | :--- |
| 起始行 | `NNN-<text>`（三位码紧跟连字符，无空格） |
| 中间行 | 可以是 `NNN-`、`NNN ` 之外的**任意文本**（RFC 959 允许，故**不能靠"行首三位数字"判断结束**） |
| 结束行 | **必须以 `NNN<SP>` 开头**（数字 + 空格）才标志多行应答终止 |
| 客户端陷阱 | 解析 `LIST` 数据时同样要小心：数据信道内容不是应答，不受此规则约束 |

> [!TIP] 排障速判
> - `220` 后立刻 `530`：服务端配置为**匿名关闭**或账号不存在（也可能 `userlist_enable` 黑名单）。
> - `PASV → 227` 后卡住再到 `425`：被动端口段没放行 / 服务端 `pasv_address` 配了内网地址。
> - `226` 但本地文件比 `SIZE` 小：中间设备截断（§1.2 的 EOF 语义命门）。
> - 只有 `550`：十有八九是权限或 chroot 边界，而不是协议问题。

---

## 🔀 5. 主动模式 vs 被动模式（核心）

### 5.1 主动模式（ACTIVE，`PORT`）建立步骤

**方向：控制由客户端发起，数据由服务端发起。**

```text
①  客户端 → 服务器:21        TCP SYN                        控制连接建立
②  服务器 → 客户端:          220 Service ready
③  客户端:                  本地随机开监听端口，例如 192.0.2.7:51001，listen()
④  客户端 → 服务器:21        PORT 192,0,2,7,199,57          （199*256+57 = 51001）
⑤  服务器 → 客户端:21        200 PORT command successful
⑥  客户端 → 服务器:21        RETR big.iso
⑦  服务器 → 客户端:21        150 Opening BINARY mode data connection for big.iso
⑧  服务器:20 → 客户端:51001  TCP SYN                        ← 服务端反向连客户端！
⑨  数据在 ⑧ 的连接上流出，传完由服务端 close()
⑩  服务器 → 客户端:21        226 Transfer complete.
```

| 关键约束 | 影响 |
| :--- | :--- |
| 服务端从 **20 端口**发起（`connect_from_port_20=YES`） | 客户端防火墙须允许"来自 20 的入站"，或使用 ESTABLISHED/RELATED 状态放行 |
| `PORT` 里写的是**客户端自己的 IP:Port** | 客户端在 NAT 后 → 写出去的是私网地址，服务端**永远连不到** → `425` |
| 客户端侧要开入站端口 | 企业桌面防火墙默认**拒绝入站** → 主动模式几乎必挂 |

### 5.2 被动模式（PASSIVE，`PASV`）建立步骤

**方向：两条连接都由客户端发起。**

```text
①  客户端 → 服务器:21        控制连接（同上）
②  客户端 → 服务器:21        PASV
③  服务器 → 客户端:21        227 Entering Passive Mode (203,0,113,10,195,80)
                            ← 服务器在 203.0.113.10 上开了监听端口 195*256+80 = 50000
④  客户端 → 服务器:50000     TCP SYN                        ← 客户端主动连服务器高端口
⑤  客户端 → 服务器:21        RETR big.iso
⑥  服务器 → 客户端:21        125 Data connection already open; transfer starting.
⑦  数据流出，传完由服务端 close()
⑧  服务器 → 客户端:21        226 Transfer complete.
```

> [!NOTE] ④ 与 ⑤ 的先后顺序实现相关
> 规范上客户端可在收到 `227` 后先连数据端口、也可先发 `RETR` 再连；多数客户端**先发命令再连**或并发进行。看到抓包里"数据连接在命令之前"不必惊讶，但**服务端在收到传输命令前不得开始写数据**，这也是 `225`/`125` 存在的原因。

### 5.3 逐步对照表

| 步骤 | 主动（PORT） | 被动（PASV） |
| :--- | :--- | :--- |
| 谁开监听 socket | **客户端**（本地高端口） | **服务器**（高端口） |
| 谁发起数据 TCP | **服务器 20 → 客户端** | **客户端 → 服务器随机高端口** |
| 端口信息传递方向 | 客户端在 `PORT` 中告知服务器 | 服务器在 `227` 中告知客户端 |
| 首条传输相关应答 | `150`（即将反向连接） | `227` + `125`/`150` |
| 客户端防火墙要求 | **需要允许入站连接**（难点） | 只需出站（友好） |
| 服务器防火墙要求 | 只需出站 | **需要放行一整段高端口入站**（配置难点） |
| NAT 友好性 | 差（私网 IP 出现在 `PORT` 中） | 较好（出站被 NAT 跟踪，只需改写 `227` 里的地址） |
| 典型故障码 | `425`（服务端连不上客户端） | `425` / 连上后 `426`（地址错、端口段未放行） |
| 多文件传输 | 每文件重发 `PORT`+新建一条数据连接 | 每文件重发 `PASV`+新建一条数据连接 |

### 5.4 为什么现实中 PASV 更常用

1. **客户端侧不可控**：办公网/家庭网的客户端在 **NAT 之后**且终端防火墙默认拒绝入站，主动模式的服务端回连必然被丢；被动模式全部连接由客户端发起，属于 NAT/状态防火墙"已建立的出站会话"，天然放行。
2. **服务端侧可运维**：服务器管理员可以自主开通入站端口段，而客户端网络管理员无法为任意 FTP 服务器开洞。
3. **可被 ALG 与云 LB 处理**：NAT ALG 与云平台安全组对"PASV 出站+高端口"的组合有成熟支持，反向连入则需额外规则。
4. **代价可接受**：被动模式把难度转移到服务端——必须**显式配置 `pasv_min_port`/`pasv_max_port` 与 `pasv_address`**（§7.3），端口段越宽，安全组越难收紧。

> [!CAUTION] 被动模式并没有"解决"双通道问题，只是把问题搬到服务器
> `227` 应答体里嵌的仍是**服务器自己认为的 IP**。云主机/容器/NAT 网关后面，这个地址常常是内网 IP（如 `10.0.2.15`），客户端照连必然超时。必须配 `pasv_address=<公网 IP>` 或依赖 `nf_conntrack_ftp` 这类 ALG 改写载荷——**而 ALG 只认明文，一旦上了 FTPS 就失效**，这是 FTPS 排障的高频根因。

### 5.5 EPRT / EPSV：为 IPv6 与 NAT 打的补丁（RFC 2428）

| 项目 | `PORT`/`PASV`（RFC 959） | `EPRT`/`EPSV`（RFC 2428） |
| :--- | :--- | :--- |
| 地址族 | 只能 IPv4（点分十进制六段） | `<n>` 指明：`1` = AF_INET，`2` = AF_INET6 |
| 分隔符 | 逗号写死 | 客户端自选（命令首字符），避开 IPv6 的 `:` |
| 端口表示 | 两个 8-bit 字节拆分 | 直接十进制端口，无拆分歧义 |
| 应答 | `227 (h1,h2,h3,h4,p1,p2)` | `229 Entering Extended Passive Mode (\|\|\|50000\|)` **不含 IP**，客户端复用控制连接目的地址 |
| NAT 友好 | `227` 内嵌地址常错 | `229` 不带地址，**天然规避地址改写问题** |

```text
C: EPSV
S: 229 Entering Extended Passive Mode (|||50000|)
C: (连接 203.0.113.10:50000 —— 与控连同址)
C: EPSV ALL                      ← 声明本会话只用 EPSV（RFC 2428）
```

> [!TIP] 选型建议
> 双栈/纯 IPv6 服务器优先 `EPSV`+`EPRT`（`FEAT` 里能看到 `EPSV EPRT ALL`）。纯 IPv4 内网且客户端老旧时再退回 `PASV`/`PORT`。若两端都支持 `EPSV`，它比 `PASV` 少一个"内嵌地址被 NAT 写错"的故障面。

---

## ⏱️ 6. 一次完整会话文本时间线

场景：客户端 `192.0.2.7` 从 `ftp.example.com`（203.0.113.10）以**被动 + 二进制**方式下载 `report.pdf`。`C>` 为客户端发送，`S<` 为服务端发送；标 `*` 的是**另一条 TCP 连接**。

```text
   ┌── 控制连接：192.0.2.7:51772 → 203.0.113.10:21 ─────────────────────────────┐
C> TCP SYN                                                          ① 建立控连
S< TCP SYN-ACK / C> ACK
S< 220 ftp.example.com: FTP server ready (vsFTPd 3.0.5).            ② 服务欢迎
C> USER alice
S< 331 Please specify the password.                                 ③ 要口令
C> PASS s3cr3t
S< 230 Login successful.                                            ④ 进入 LOGIN 态
C> TYPE I
S< 200 Switching to Binary mode.                                    ⑤ 图片/二进制模式
C> PWD
S< 257 "/" is the current directory.
C> PASV
S< 227 Entering Passive Mode (203,0,113,10,195,80).                 ⑥ 数据端口 = 50000
   │   数据连接：192.0.2.7:51773 → 203.0.113.10:50000  *
C> TCP SYN ... (三次握手完成)                                        ⑦ 客户端反向连接就绪
C> RETR report.pdf
S< 150 Opening BINARY mode data connection for report.pdf (482190 bytes).   ⑧
S< ===== 数据连接上推送 482190 字节原始文件内容（无任何协议头） =====         ⑨
S< * 数据连接由服务端 close() → 客户端收到 EOF，判定传输结束
S< 226 Transfer complete.                                            ⑩ 控连上收尾
C> SIZE report.pdf
S< 250 482190                                                        ⑪ 完整性复核
C> QUIT
S< 221 Goodbye.                                                      ⑫ 控连关闭
   └──────────────────────────────────────────────────────────────────────────┘
```

### 6.1 阶段解读

| # | 阶段 | 要点 |
| :--- | :--- | :--- |
| ①② | 控制连接 + `220` | **服务端主动推 `220`**，客户端未发任何字节；这是 FTP 与 HTTP 最大的交互差别（服务器先说话） |
| ③④⑤ | 登录 + `TYPE I` | 口令必须单独一行且明文，失败 `530`；`TYPE` 不建数据连接，只改会话状态，漏发可能默认 `A` 而改坏二进制文件 |
| ⑥⑦ | `PASV`→`227`→客户端连 50000 | 数据连接由**客户端**发起（被动模式定义） |
| ⑧⑨⑩ | `RETR`→`150`→裸字节流→`226` | 数据信道上**只有文件内容**（`150` 文本里的大小是提示，非协议保证）；`226` 在关闭数据连接之后才发，**收到 `226` ≠ 内容完整**（见 §1.2） |
| ⑪⑫ | `SIZE` 复核、`QUIT`→`221` | 生产脚本必做 `SIZE`/hash 复核（`MDTM` 可比时间戳）；不发 `QUIT` 直接断开也算合法结束，但服务端会记异常会话并可能回 `421` |

### 6.2 会话时延与连接开销估算

$$
\text{RTT}_{\text{session}} \;\approx\; \underbrace{1}_{\text{控连 TCP}} + \underbrace{2}_{\text{USER/PASS}} + \underbrace{2}_{\text{TYPE/PASV}} + \underbrace{1}_{\text{数据 TCP}} + \underbrace{2}_{\text{RETR/226}} + \underbrace{\frac{S}{BW}}_{\text{传输}}
$$

| 项目 | 数量（N 个文件，单会话） | 优化手段 |
| :--- | :--- | :--- |
| TCP 握手 + 锁步 RTT | $1 + N$ 次握手，每命令 1 次往返 | 握手无法合并（每文件一条数据连接）；用 `lftp -j N` 多会话并行、`xargs -P` 多进程，或直接换 HTTP/SFTP |
| 慢启动 + TLS（FTPS） | 每条数据连接重新慢启动；控连 1~2 RTT 且**每条数据连接再握手一次** | 大文件友好、海量小文件极差 → 打包后传；配会话票据/`require_ssl_reuse` 复用 TLS 会话 |

> [!WARNING] FTP 不适合"海量小文件"
> 每个文件都要付一条新 TCP 连接 + 一轮慢启动 + 若干锁步 RTT。传 10 万个 4 KB 文件，握手开销远超数据本身。工程做法：**先打包再传**，或换 rsync/HTTP2/SFTP。

---

## 📶 7. 断点续传、匿名 FTP 与服务器配置

### 7.1 REST：断点续传（Restart）

```text
C> PASV
S< 227 Entering Passive Mode (203,0,113,10,195,80).
C> REST 418190                     ← 从第 418190 字节处继续
S< 350 Restarting at 418190. Send STORE or RETRIEVE to initiate transfer.
C> RETR big.iso                    ← REST 必须"紧贴"一条传输命令
S< 150 Opening BINARY mode data connection for big.iso.
```

| 细节 | 说明 |
| :--- | :--- |
| 标记单位 | `TYPE I` 下 = **字节偏移**（RFC 3659 的 stream-mode restart marker，十进制）；`TYPE A` 下 = **记录/行数**（959 遗留，实现不一） |
| 与传输命令的配对 | `REST` 只设状态，**必须紧随 `RETR`/`STOR`/`APPE`**，否则被后续命令清掉；顺序错会得 `503 Bad sequence of commands` |
| 续传方向 | **下载**：客户端按本地已存大小 `local_size` 发 `REST local_size`，服务端从该偏移继续写，配合 `SIZE` 判断是否完整；**上传**：更常用 `APPE` 直接追加，或对服务端残留的部分文件 `REST`+`STOR`（vsftpd 可配隐藏临时名） |
| 一致性风险 | 若服务端文件在断点之后**被替换过**，续传拼出的内容不可信 → 先比对 `MDTM`/hash 再决定续传还是重传；`FEAT` 中出现 `REST STREAM` 才表示支持流模式续传 |

### 7.2 匿名 FTP（Anonymous FTP）

| 要素 | 约定 |
| :--- | :--- |
| 用户名 / 口令 | 用户名 `anonymous`（多数实现也接受 `ftp`）；口令惯例为**用户邮箱**（RFC 1635），现代服务器常忽略内容、只要求非空 |
| 权限模型 | 服务端把该会话映射到**低权限账号 `ftp`**，chroot 到 `/srv/ftp`，通常**只读** |
| 上传目录 | 若开放需**单独目录 + 全局可写 + 禁止下载**（典型 `upload/`，文件权限 `0666`，`chmod 0644` 后由人工分拣） |
| 应答与现况 | 成功可能回 `230-Welcome` 多行公告 + `230`，未开匿名则 `530 Login incorrect`；公开镜像站仍在用，但**匿名 FTP 已成扫描与滥用靶点**（跳板/恶意文件投递），对外建议改 HTTPS 目录浏览或 SFTP |

```text
C> USER anonymous
S< 331 Please specify the password.
C> PASS client@example.com
S< 230-Welcome to the example mirror.
S< 230 Login successful.
```

### 7.3 被动端口范围配置（部署必做）

**问题**：`PASV` 默认让服务端从"任意临时端口"里挑一个监听，安全组只能"全开"或"不开"。
**解法**：把随机端口收敛到一个**显式、有限、可放行**的区间。

```ini
# /etc/vsftpd/vsftpd.conf（代表性片段）
anonymous_enable=NO            # 非必要就关掉匿名
local_enable=YES
write_enable=YES
chroot_local_user=YES
allow_writeable_chroot=YES

pasv_enable=YES                # 强制被动
pasv_min_port=50000            # ← 端口段下界
pasv_max_port=50050            # ← 上界：区间宽度 ≥ 预期并发数（每连接占 1 个）
pasv_address=203.0.113.10      # 填公网 IP（NAT/云主机后面必须填，否则 227 报内网地址）
connect_from_port_20=YES       # 若仍允许 PORT，主动模式数据源端口用 20

idle_session_timeout=300       # 空闲 5 min 断开（配合客户端 NOOP 保活）
data_connection_timeout=120    # 数据连接无活动的超时 → 常表现为 425/426
accept_timeout=60              # 等待客户端连上被动端口的超时

ssl_enable=YES                 # 转 FTPS（RFC 4217）
allow_anon_ssl=NO
force_local_logins_ssl=YES     # 口令必须加密
force_local_data_ssl=YES
require_ssl_reuse=YES          # 数据连接须复用控连会话（RFC 4217 要求）
```

> [!NOTE] 其他服务端等价指令
> ProFTPD：`PassivePorts 50000 50050` + `MasqueradeAddress 203.0.113.10` + `DefaultAddress 10.0.0.5`；FileZilla Server：Settings → FTP → "Custom port range" + "Use my external IP"。

```bash
# 防火墙：21 + 收敛后的被动段
iptables -A INPUT -p tcp --dport 21 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 50000:50050 -m conntrack --ctstate NEW -j ACCEPT
# ALG（只对明文 FTP 有效，FTPS/SFTP 载荷加密，改写不了）
modprobe nf_conntrack_ftp
echo 'nf_conntrack_ftp' > /etc/modules-load.d/ftp.conf
# 服务器主动连出（若仍支持 PORT）：客户端 IP 白名单极难维护，建议直接禁 PORT
```

| 配置要点 | 常见踩法 |
| :--- | :--- |
| 端口段宽度 | 太窄 → 并发一多就 `425`/连不上；太宽 → 攻击面与扫描噪音。按**峰值并发 × 1.5** 估 |
| `pasv_address` | 忘记配 → `227` 返回内网 IP，客户端在公网连私网地址超时（表现是长时间 hang 后 `425`） |
| 依赖 ALG | 只在内网明文 FTP 可行；上 FTPS 后**必须**关 ALG 并正确配 `pasv_address` |
| 主动模式 | 生产建议**只留 PASV**（禁 `PORT`），减少"服务端向任意地址连出"的滥用面（§8.2） |
| 云 LB | LVS/CLB 四层透传可用；**七层 LB 不支持 FTP 双通道**，需 NLB/主机端口映射 |

### 7.4 客户端实操速查

```bash
# 交互式：ftp -p ftp.example.com（-p = 被动模式，多数系统默认已开）
curl -sS --ftp-ssl-reqd -u alice:*** ftp://ftp.example.com/report.pdf -o report.pdf
curl -T build.bin -u alice:*** --ftp-create-dirs ftp://ftp.example.com/incoming/
curl -C - -o big.iso ftp://ftp.example.com/big.iso     # -C - 自动续传（按本地大小发 REST）

# lftp：镜像 / 并发 / 重试
lftp -u alice,*** ftp://ftp.example.com -e 'set net:max-retries 3; mirror -c --parallel 4 /logs /tmp/logs; quit'
```

> [!TIP] 抓包定位一句话
> `tcpdump -i any -nn -s0 'port 21 or (tcp[tcpflags] & tcp-syn != 0 and host 203.0.113.10)' -w ftp.pcap`
> 只看 21 端口会漏掉数据连接；必须把 `227` 里解出的高端口一起抓。Wireshark 里 `Decode As → FTP` 只解明文，FTPS 要先喂密钥。

---

## 🔒 8. 安全风险与缓解

### 8.1 明文协议的根本问题

| 风险 | 机理 | 后果 / 缓解 |
| :--- | :--- | :--- |
| **凭据嗅探** | `USER`/`PASS` 以明文行出现在 21 端口载荷中 | 一次抓包即得账号；缓解：`force_local_logins_ssl=YES`（FTPS）或直接换 SFTP |
| **内容窃听/篡改** | 数据信道无任何机密性与完整性保护，仅 TCP 校验和 | 中间人可改文件、塞后门；FTP **不提供任何校验机制**，`226` 不代表内容可信 → 应用侧强制 hash/签名 |
| **会话劫持** | 攻击者若能伪造 `PORT`/预测序列号，可插入数据连接 | 现代内核端口随机化 + `421` 超时收敛风险；根治靠 TLS/SSH |
| **重放与枚举** | `SIZE`/`MDTM`/`RNFR` 的应答差异可枚举文件名与目录结构 | 匿名 FTP 尤其明显；关闭无关扩展、限制匿名可见目录 |
| **日志与凭据外泄** | 命令行内联口令（`ftp -n <<EOF` 里的 `PASS`）、`curl ftp://user:pass@host` | 进 shell history 与进程列表 → 用 `_netrc`（权限 600）或环境注入，禁内联 |
| **暴力破解与降级** | 21 端口长期暴露且无速率限制；客户端"能明文就连明文" | 服务端 `fail2ban`/连接速率限制/账号锁定；客户端强制 TLS：`AUTH TLS` 失败即断开、禁止回退（对齐 RFC 4217 §4/§5） |

### 8.2 双通道特有攻击面

| 攻击 | 手法 | 缓解 |
| :--- | :--- | :--- |
| **FTP Bounce / 第三方 PORT** | 客户端把 `PORT` 指向**受害者 IP:端口**，让服务器代其发起连接，用于端口扫描或把文件投递到第三台服务器（配合 `STOU`+`RETR` 形成"文件走私"） | 服务端校验 `PORT` 地址必须等于控制连接对端（ProFTPD `PortPromiscuous off` / vsftpd 默认行为）；生产禁主动模式；开启 `EPRT` 同类校验 |
| **向任意高端口连出** | 主动模式让服务器沦为内网扫描器（打到防火墙内网侧） | 出方向 ACL 限制服务器主动连接的目标；禁 `PORT` |
| **`227`/ALG 载荷改写** | 中间设备改写嵌入地址，配置错误即流量错位 | 明确 `pasv_address`，云环境关闭/避免依赖 ALG |
| **资源耗尽（列表放大 / 半开会话）** | 百万文件目录的 `LIST`/`MLSD` 持续占满数据连接与 fd；只完成控制握手不发命令占住会话槽位 | 目录分片、`max_per_ip`/`max_clients`、`idle_session_timeout`、`data_connection_timeout`、被动段限流与 `xferlog` 审计 |

### 8.3 迁移决策

| 场景 | 推荐 | 理由 |
| :--- | :--- | :--- |
| 内网设备只支持裸 FTP | 隔离 VLAN + 静态口令短周期 + 禁 PORT + 只开必要目录 | 承认无法加密，用网络边界兜底 |
| 人工/批量站点发布 | **SFTP**（[[SSH]]）或 FTPS | 单端口、密钥认证、审计统一 |
| 面向浏览器/公网分发、需断点续传 | **HTTPS 对象存储/CDN + `Range` 请求**（[[http]]） | 无 NAT 双通道问题、可缓存、可签名鉴权；`Range` 比 FTP `REST` 更可控可观测 |
| 大文件长期同步 | rsync/SFTP 或专用传输（Aspera 类） | FTP 无增量语义 |

> [!CAUTION] FTPS 也别踩两个坑
> ① **`require_ssl_reuse`**：RFC 4217 要求数据连接的 TLS 会话复用控制连接的会话参数，否则许多服务端拒绝——客户端不支持复用会表现为"登录成功但 `425`/`530`"。② **显式 FTPS 的端口仍是 21**，与裸 FTP 同端口，运维上必须明确"这台机器只允许哪种"，否则出现用户混用 `ftp://` 与 `ftpes://` 导致一半会话明文。

---

## 🖼️ FTP 会话与主动/被动时序图

![[FTP.excalidraw]]

---

## 🔗 思考与延伸索引

- [[TCP]] — FTP 的承载层：控制/数据两条连接各自的三次握手与 FIN 语义，"数据结束 = 连接关闭"依赖的正是 TCP 的有序可靠交付
- [[SSH]] — SFTP 的承载层：单端口多路复用（channel）彻底避开双通道 NAT 问题，对照理解 FTP 的历史包袱
- [[http]] — 另一条路：`Range` 断点续传、持久连接多路复用、无带内地址，是 FTP 现代替代方案（HTTPS/CDN 分发）的技术基础
- [[TLS]] — FTPS（RFC 4217）的密码学层；也是"ALG 失效"与"数据连接会话复用"两个坑的根源
- [[IP]] / [[UDP]] — `PORT`/`227` 里嵌的就是 IP 层地址（IPv6 引出 EPRT/EPSV）；对照组 UDP 型协议（如 TFTP）没有"连接切换"概念，带内地址表现完全不同
- [[mqtt]] — 同为"会话式应用层协议"，但用单连接 + 报文类型表达状态，可对照 FTP 的锁步文本命令

---

## 📚 参考资料

1. **IETF** — *RFC 959: File Transfer Protocol (FTP)*（J. Postel, J. Reynolds, 1985：双通道模型、命令与应答码全集、NVT-ASCII 与 TYPE/STRU/MODE，附录中的典型会话文本与 server/user-FTP 主动/被动进程交互）.
2. **IETF** — *RFC 4217: Securing FTP with TLS*（显式 FTPS：`AUTH TLS`/`PBSZ`/`PROT`、会话复用要求、禁止明文回退）.
3. **IETF** — *RFC 2428: FTP Extensions for IPv6 and NATs*（`EPRT`/`EPSV`、`229` 应答、`EPSV ALL`、可扩展定界符）.
4. **IETF** — *RFC 3659: Extensions to FTP*（`SIZE`/`MDTM`/`MLSD`/`MLST`、流模式 restart marker）.
5. **IETF** — *RFC 2228: FTP Security Extensions*（`AUTH`/`PROT` 框架，FTPS 的上位规范）；*RFC 2389: Feature negotiation mechanism for the File Transfer Protocol*（`FEAT`/`OPTS`）.
6. **IETF** — *RFC 2640: Internationalizing the File Transfer Protocol*（`OPTS UTF8 ON` 与语言协商）；*RFC 1635: How to Use Anonymous FTP*；*RFC 2577: FTP Request For Comments Frequently Asked Questions*.
7. **IANA** — *Service Names and Transport Protocol Port Numbers*（`ftp/21`、`ftp-data/20`、`ftps/990`、`ftps-data/989`）.
8. **Linux 文档** — *vsftpd.conf(5)*（`pasv_min_port`/`pasv_max_port`/`pasv_address`/`require_ssl_reuse`/`allow_writeable_chroot`）、*ProFTPD Configuration Directives*（`PassivePorts`/`MasqueradeAddress`）.
9. **curl / lftp 手册** — *curl FTP 协议选项*（`--ftp-create-dirs`、`-C -`、`--ftp-ssl`、EPSV 行为）、*lftp mirror 手册*；**W. R. Stevens** — *TCP/IP Illustrated, Vol.1*（半关闭语义）；**B. Chin, S. Hanks** — *Linux Firewalls*（FTP ALG 与 conntrack 陷阱）.
