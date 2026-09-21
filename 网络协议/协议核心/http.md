---
title: HTTP 协议完整技术手册
tags:
  - Network
  - Protocol
  - HTTP
  - HTTPS
  - Web
  - REST
status: Completed
created: 2026-09-21
---

# 🌐 HTTP 协议完整技术手册

> [!INFO] 笔记概览
> - **协议定位**：应用层（Application Layer）的**请求-响应**协议，本身**无状态（stateless）**；一条逻辑连接承载于 [[TCP]]（HTTP/3 承载于 QUIC/UDP），可再套一层 TLS 成为 HTTPS。协议只规定报文语法与语义，不关心正文内容（HTML/JSON/图片皆可）。
> - **内容范围**：版本演进 · 报文格式 · 方法语义 · 状态码 · 关键头部 · URL 结构 · Cookie/会话 · HTTPS 与 TLS 的关系 · 一次完整请求时序
> - **推荐路径**：§1 定位与版本 → §2 报文格式 → §3 方法 → §4 状态码 → §5 头部 → §6 URL → §7 会话保持 → §8 HTTPS → §9 完整时序 → §10 图

---

## 🧭 1. 协议定位与版本演进

### 1.1 定位要素

| 属性 | 说明 |
| :--- | :--- |
| 协议全称 | HyperText Transfer Protocol（超文本传输协议） |
| 所在层次 | 应用层（OSI 第 7 层 / TCP-IP 应用层） |
| 通信模型 | **请求-响应**（Request-Response）：客户端发起，服务端应答；服务端不能主动推送（需 WebSocket/SSE 等扩展） |
| 状态特性 | **无状态**：协议本身不记录前后请求关系；会话靠 Cookie/Session/Token 在应用层补齐，见 §7 |
| 下层承载 | HTTP/1.x、HTTP/2 → [[TCP]]（80/443）；HTTP/3 → QUIC（**UDP** 7654/443） |
| 报文可读性 | HTTP/1.x 为**纯文本**协议（可 telnet/curl 手测）；HTTP/2 为**二进制分帧**，不可直读 |
| 语义规范 | 方法、状态码、头部字段的语义由 RFC 9110 定义；报文语法与连接管理由 RFC 9112 定义 |
| 典型扩展 | Range 断点续传、Content-Encoding 压缩、Conditional 缓存、Pipeline、多路复用 |

### 1.2 版本演进对照表

| 版本 | 年份 | 传输 | 连接 | 关键差异 | 现状 |
| :--: | :--: | :--- | :--- | :--- | :--- |
| **HTTP/0.9** | 1991 | TCP | 短连接 | 只有 `GET /file` 一行请求，**无头部、无状态码**，响应为 ASCII 超文本，传完即断 | 已废弃 |
| **HTTP/1.0** | 1996 | TCP | 默认**每请求一连接** | 引入请求行/状态行/头部/状态码、多方法、MIME `Content-Type`；可用 `Connection: keep-alive` 试验性保活 | 兼容模式 |
| **HTTP/1.1** | 1997（RFC 9112 修订 2022） | TCP | **持久连接（Persistent）默认开启** | `Host` 头部**强制必需**（虚拟主机）；`Transfer-Encoding: chunked` 分块传输；**管线化 Pipelining**（实际因队头阻塞少用）；Range、`Cache-Control`、`Connection: close` 明确化 | 部署最广 |
| **HTTP/2** | 2015（RFC 7540 → 9113） | 单条 TCP | 长连接 | **二进制分帧层**：请求/响应拆为 Frame → Message → Stream；一条连接**多路复用**并发多个请求，解决应用层队头阻塞；`HPACK` 头部压缩；服务端推送（Push，实际少用）、流优先级 | 主流（多经 TLS） |
| **HTTP/3** | 2022（RFC 9114） | **QUIC over UDP** | 连接 = 四元组 + 连接 ID | 传输层集成 TLS 1.3，**0-RTT/1-RTT 建连**；流独立重传，彻底消除 TCP 层队头阻塞；**连接迁移**（换网络不断线，用 Connection ID 而非四元组标识） | 快速增长 |

> [!WARNING] 版本与首行的对应关系
> 报文首行仍写 `HTTP/1.1` 是 HTTP/2 的**语义兼容**表象：H2 通过分帧层与 ALPN 协商版本，`HTTP/2.0` 形式的请求行**不会出现在线上**（HTTP/1.1 语法被映射进 HEADERS 帧的伪头部 `:method`/`:path`/`:status`）。升级靠 ALPN 协商，不靠改写版本字符串。

---

## 📦 2. 报文格式（Message Syntax）

### 2.1 三段式通用结构

任何 HTTP/1.x 报文都由 **起始行 + 头部 + 空行 + 可选正文** 构成，**行分隔符固定为 CRLF（`\r\n`，CR=0x0D、LF=0x0A）**：

```text
请求报文                                        响应报文
──────────────────────────────────             ──────────────────────────────────
POST /api/login HTTP/11 ← 请求行                HTTP/1.1 200 OK ← 状态行
Host: example.com        ┐                     Date: Fri, 21 Sep 2026 …  ┐
Content-Type: application/json │ 头部字段 ×N   Content-Type: application/json │ 头部字段 ×N
Content-Length: 27       ┘                     Content-Length: 132        ┘
                             ← 空行 CRLF                                    ← 空行 CRLF
{"u":"alice","p":"1234"}   ← 正文（可选）      {"code":0,"msg":"ok"}      ← 正文（可选）
```

| 组成 | 请求报文 | 响应报文 | 规则 |
| :--- | :--- | :--- | :--- |
| **起始行** | 请求行 `method SP request-target SP HTTP-version` | 状态行 `HTTP-version SP status-code SP reason-phrase` | 均以 CRLF 结束；只允许 1 行 |
| **头部（Headers）** | `field-name ":" OWS field-value OWS` | 同 | 每字段一行，冒号后空白可忽略；**字段名大小写不敏感**，值大小写敏感；禁止裸 CR/LF（防响应拆分/请求走私） |
| **空行** | 单独一个 CRLF | 同 | 标志头部结束；即使无正文也必须存在（"无正文"即紧跟空行） |
| **正文（Body / Content）** | 可选（POST/PUT 常见） | 可选（204/304 必须无正文） | 长度由 `Content-Length` 或 `Transfer-Encoding: chunked` 界定 |

- **状态行短语（Reason-Phrase）仅为人类可读提示**，客户端判定以**数字状态码**为准，可省略。
- **obsolete line folding**（头部值折行，行首空格续接）在 HTTP/1.1 中已废弃，收到应拒绝。
- HTTP/2、HTTP/3 线上形态为二进制帧（Frame），语义仍映射到"起始行 + 头部 + DATA 正文"。

---

## 🛠️ 3. 请求方法（Request Methods）

**安全（Safe）**＝不改变服务器状态，仅读取，可被预取/缓存；**幂等（Idempotent）**＝同一请求重复 N 次的副作用与 1 次相同。

| 方法 | 语义 | 安全 | 幂等 | 正文 | 要点 |
| :--- | :--- | :--: | :--: | :--: | :--- |
| **GET** | 读取指定资源表示（representation） | ✅ | ✅ | 一般无 | 语义上参数在 query；响应可缓存；不应承载敏感数据（URL 会进日志/Referer） |
| **POST** | 提交实体，由服务器决定如何处理（创建/执行） | ❌ | ❌ | 有 | 非幂等：重复提交会重复下单，需幂等键（Idempotency-Key）在应用层解决 |
| **PUT** | 用请求正文**整体替换**目标资源（客户端决定 URI） | ❌ | ✅ | 有 | 重复 PUT 结果一致；创建也常用 PUT（已知 URI） |
| **PATCH** | 对资源做**部分**修改 | ❌ | ❌* | 有 | 幂等性取决于补丁实现（默认非幂等）；补丁格式如 JSON Patch / JSON Merge Patch |
| **DELETE** | 删除目标资源 | ❌ | ✅ | 通常无 | 重复 DELETE 通常第二次返回 404/410，仍算幂等（无额外副作用） |
| **HEAD** | 与 GET 相同，但**响应只给头部无正文**（必须无 `Content-Length` 之外的正文） | ✅ | ✅ | 无 | 探测资源存在性/大小/最后修改时间，省带宽；缓存校验常用 |
| **OPTIONS** | 查询目标资源的可用方法/通信选项 | ✅ | ✅ | 无 | CORS 预检（Preflight）即 `OPTIONS` + `Access-Control-Request-Method` |
| **TRACE** | 回显收到的请求，用于诊断链路 | ✅ | ✅ | 无 | 有 XST（跨站追踪）风险，服务器普遍禁用（返回 405） |
| （扩展） | `CONNECT`：与代理建立透明隧道，HTTPS 过代理的关键 | ❌ | ❌ | 无 | `CONNECT host:443 HTTP/1.1` → 响应 `200 Connection established` 后转二进制隧道 |

> [!NOTE] 方法名大小写
> 方法为**区分大小写**的令牌（token），必须写 `GET` 而非 `get`。自定义方法属于扩展，不被通用实现理解，REST 设计应避免。

---

## 🚦 4. 状态码（Status Codes）

### 4.1 五类总览

| 类别 | 含义 | 通用语义 | 可否缓存 |
| :--: | :--- | :--- | :--- |
| **1xx** | Informational（信息性） | 请求已接收、继续处理；只起始行 + 头部，无正文 | 否（101 升级后除外） |
| **2xx** | Successful（成功） | 请求被正常接收、理解、接受 | 视 `Cache-Control` |
| **3xx** | Redirection（重定向） | 需采取进一步动作（多为换 URI）；带 `Location` | 视指令 |
| **4xx** | Client Error（客户端错误） | 请求语法/语义/权限有问题，责任在客户端 | 默认可缓存（404 有期限） |
| **5xx** | Server Error（服务端错误） | 请求合法但服务器无法完成 | 默认不缓存 |

### 4.2 常用码逐个说明

| 状态码 | 名称 | 含义与典型处理 |
| :--: | :--- | :--- |
| **200** | OK | 请求成功，正文即资源表示；GET/POST/PUT 最常见应答 |
| **201** | Created | 资源已创建（POST 建资源 / PUT 建资源），响应应带 `Location: /orders/42` 指向新资源 |
| **204** | No Content | 成功但**响应无正文**（`DELETE` 后或只更新头部）；浏览器保留当前文档，不得用于要求正文的场景 |
| **206** | Partial Content | `Range` 请求成功，正文为指定字节段，可带 `Content-Range` 或多段 `multipart/byteranges` |
| **301** | Moved Permanently | **永久**重定向：URL 已失效，客户端/搜索引擎可更新书签并缓存新地址（POST→GET 视实现，谨慎） |
| **302** | Found | **临时**重定向：不改写原 URL；历史上允许 POST 转 GET（浏览器实际行为） |
| **307** | Temporary Redirect | 临时重定向且**禁止改变方法**（POST 仍发 POST） |
| **308** | Permanent Redirect | 永久重定向且**禁止改变方法**（对应 301 的严格版） |
| **304** | Not Modified | 条件请求命中缓存：**只给头部不给正文**，客户端续用本地副本（须带 `ETag`/`Last-Modified` 校验） |
| **400** | Bad Request | 请求语法错误/字段校验失败/无法解析正文（如非法 JSON、百分号编码残缺） |
| **401** | Unauthorized | **未认证**（身份没证明）：响应必须带 `WWW-Authenticate` 质询头 |
| **403** | Forbidden | **未授权**（身份已明、权限不足）或方法/来源被拒；重发同请求不会成功 |
| **404** | Not Found | 资源不存在（也可能是故意隐藏存在性，避免泄露）；无副作用 |
| **405** | Method Not Allowed | URI 存在但方法不支持，应带 `Allow: GET, HEAD` 列出可用方法 |
| **408** | Request Timeout | 服务端等客户端发完请求超时（TCP 已连、正文迟迟不来），可重试 |
| **409** | Conflict | 与资源当前状态冲突（重复创建、乐观锁版本不一致、并发 PUT） |
| **413** | Content Too Large | 正文超过服务器限制（`max_body_size`） |
| **429** | Too Many Requests | 限流：应带 `Retry-After: 30` 提示重试时间；客户端须退避（指数退避 + 抖动） |
| **431** | Request Header Fields Too Large | 头部/请求行过大，需收缩 Cookie 或头部数量 |
| **500** | Internal Server Error | 服务端未处理异常/程序错误；不可盲目重试（可能已产生副作用） |
| **502** | Bad Gateway | 网关/反向代理从**上游**收到无效应答（上游进程崩溃、协议不匹配） |
| **503** | Service Unavailable | 过载或维护中暂不可用；带 `Retry-After` 时应退避重试，可配 `Connection: close` |
| **504** | Gateway Timeout | 网关/代理**等待上游超时**（上游慢查询、下游线程池耗尽）；与 502 区分：慢 vs 错 |
| **505** | HTTP Version Not Supported | 不支持请求使用的协议版本 |

> [!TIP] 排障速判
> `401 vs 403` → 先确认令牌是否有效（认证），再看角色权限（授权）；`502/504` → 看**上游服务**健康与超时链（网关超时必须 > 上游超时，否则永远拿不到上游真实错误码）；`408 vs 504` → 前者是"客户端没发完"，后者是"服务端没等到上游"。

---

## 📝 5. 重要头部字段（Header Fields）

### 5.1 核心头部速查表

| 头部 | 方向 | 作用 | 示例与说明 |
| :--- | :--: | :--- | :--- |
| **Host** | 请求 | HTTP/1.1 **必需**，指定目标主机与端口，是虚拟主机（一 IP 多站点）与 `CONNECT` 的依据 | `Host: www.example.com:8443` |
| **Content-Type** | 双向 | 声明正文媒体类型（MIME）与参数 | `application/json; charset=utf-8`、`text/html; charset=utf-8`、`multipart/form-data`（文件上传）、`application/x-www-form-urlencoded` |
| **Content-Length** | 双向 | 正文字节数，接收方据此判定正文结束 | `Content-Length: 27`；与 chunked 二选一，H1 中两者同现时以 chunked 为准（易成**请求走私**漏洞面） |
| **Transfer-Encoding** | 双向 | 传输层编码（分段/压缩），**逐跳（hop-by-hop）** | `Transfer-Encoding: chunked`（长度未知/流式响应）；H2/H3 **禁止**该头部 |
| **Connection** | 双向 | 逐跳选项 | `keep-alive`（H1.1 默认）、`close`（禁用复用，代理转发前常重写）、`Upgrade: websocket` |
| **Cookie** | 请求 | 回传匹配到的 cookie（多个用 `; ` 分隔） | `Cookie: sid=abc123; theme=dark` |
| **Set-Cookie** | 响应 | 服务器下发 cookie（仅此一个响应头，属性见 §7） | `Set-Cookie: sid=abc123; Path=/; HttpOnly; Secure; SameSite=Lax` |
| **Authorization** | 请求 | 携带认证凭据 | `Bearer <token>`、`Basic base64(user:pass)`、`Digest …` |
| **WWW-Authenticate** | 响应 | 401 时声明认证方案 | `WWW-Authenticate: Bearer realm="api" error="invalid_token"` |
| **Accept / Accept-Encoding / Accept-Language / Content-Language** | 双向 | 内容协商 | `Accept: application/json`、`Accept-Encoding: gzip, br, zstd` → 响应 `Content-Encoding: br` |
| **Vary** | 响应 | 告知缓存以哪些请求头为键区分副本 | `Vary: Accept-Encoding, Origin`（CORS 与压缩同时命中必需） |
| **User-Agent / Server / Date / Referer / Origin** | 双向 | 客户端标识 / 服务端标识 / 应答时间 / 来源页 / CORS 请求源 | `Origin: https://app.example.com` 是 CORS 判定依据 |
| **Location** | 响应 | 3xx 重定向目标；201 新建资源 URI | `Location: /orders/42` |
| **Retry-After** | 响应 | 429/503 建议等待秒数或 HTTP-date | `Retry-After: 30` |
| **Allow** | 响应 | 405 时列出可用方法 | `Allow: GET, HEAD, OPTIONS` |
| **Range / Content-Range / Accept-Ranges** | 双向 | 断点续传、视频拖动、分片下载 | 见 §5.3 |
| **Cache-Control / ETag / If-None-Match / Last-Modified / If-Modified-Since / Age** | 双向 | 缓存与条件请求 | 见 §5.2 |
| **X-Forwarded-For / X-Forwarded-Proto** | 请求 | 代理链传递真实客户端 IP 与协议（事实标准，非 RFC 定义） | `X-Forwarded-For: 203.0.113.7, 10.0.0.2`（仅信任自己代理追加的最右一段） |

### 5.2 缓存与条件请求（Cache & Conditional Requests）

**指令分两层**：`Cache-Control`（RFC 9111，优先级高于旧的 `Expires`）。

| 指令 | 位置 | 语义 |
| :--- | :--- | :--- |
| `max-age=3600` | 响应 | 新鲜期 3600 s（相对接收时刻），过期后须向服务器校验 |
| `s-maxage=60` | 响应 | 只对**共享缓存（CDN）**生效的新鲜期 |
| `no-cache` | 请求/响应 | 可以用副本，但**每次必须先回源校验**（不是"不缓存"） |
| `no-store` | 请求/响应 | 任何存储都不得保留（token、PII 应答） |
| `public` / `private` | 响应 | 是否允许共享缓存保存（`private` 只允许浏览器本地） |
| `must-revalidate` / `proxy-revalidate` | 响应 | 过期后禁止在无法联系服务器时"凑合用"（stale） |
| `immutable` | 响应 | 资源永不改变，新鲜期内连校验请求都不发（配长 `max-age` 用于带指纹的静态资源） |
| `min-fresh` / `max-stale` | 请求 | 客户端可接受的新鲜度约束 |

**校验器（Validator）**：强校验 `ETag: "675af3428e0"`（配合 `If-None-Match`）；弱校验 `Last-Modified`（配合 `If-Modified-Since`，秒级精度、改回旧内容会误判，故优先 ETag）。

流程（304 命中，零正文传输）：

```text
① 首次              GET /logo.png            → 200 OK
                    Cache-Control: max-age=60
                    ETag: "a1b2"
② 60 s 内          [本地直接返回，不发请求]     cache hit
③ 超过 60 s         GET /logo.png
                    If-None-Match: "a1b2"     → 304 Not Modified
                    ETag: "a1b2"              （无正文，续用本地副本并刷新新鲜期）
```

> [!NOTE] 静态资源最佳实践
> 文件名带内容哈希（`app.3f8a2c.js`）→ 响应 `Cache-Control: public, max-age=31536000, immutable`；HTML 入口页用 `no-cache` 或短 `max-age` + ETag，避免"新资源已发布、HTML 仍缓存"的错版。

### 5.3 chunked 分块传输编码

用途：正文长度**未知或流式生成**（动态页面、SSE 前身、大文件边压缩边发）。语法（RFC 9112 §7.1）：

```abnf
Chunked-Body   = *chunk last-chunk trailer-part CRLF
chunk          = chunk-size [ chunk-ext ] CRLF chunk-data CRLF
chunk-size     = 1*HEXDIG                 ; 十六进制，不含前导 0x
last-chunk     = 1*("0") [ chunk-ext ] CRLF
chunk-ext      = ";" *( chunk-ext-name [ "=" chunk-ext-val ] )
```

线上字节流示例（三段：0x25=37、0x1a=26、0 结束块）：

```text
25\r\n
{"id":1,"name":"obsidian-note"}\r\n
1a\r\n
,"size":1024}\r\n
0\r\n
X-Checksum: 9f2c\r\n     ← 可选 trailer（尾部头部）
\r\n                      ← 结束标志：0 长度块之后必须再一个 CRLF
```

- 解码端：反复读"长度行 → 丢弃该行长度字节 → 期望一个 CRLF"，直到长度为 0。
- `chunked` 只能出现在**响应**或**有正文的请求**中；它替代 `Content-Length`，二者不应同现。
- HTTP/2/3 用帧的 `END_STREAM` 标志界定消息结束，**不再使用 chunked**。

### 5.4 Range 与部分内容

```text
请求：Range: bytes=500-999           单个 500 字节
     Range: bytes=1000-             从 1000 到结尾
     Range: bytes=-300              最后 300 字节
     Range: bytes=0-499, 1000-1499  多段
应答：206 Partial Content
     Accept-Ranges: bytes
     Content-Range: bytes 500-999/10240     ← 本段/总长
     Content-Length: 500
多段应答：Content-Type: multipart/byteranges; boundary=…
越界：416 Range Not Satisfiable；不支持：忽略 Range 直接 200 全量
```

---

## 🔗 6. URL / URI 结构

### 6.1 语法分解

通用 URI 形式（RFC 3986）：`scheme:[//authority]path[?query][#fragment]`，其中 `authority = [userinfo@]host[:port]`。

```text
https://alice:s3cr3t@www.example.com:8443/shop/list?cat=book&page=2#section-3
└─┬─┘   └──┬─────┘   └─────┬─────┘ └─┬─┘ └────┬────┘ └─────┬──────┘ └───┬───┘
scheme  userinfo            host     port      path          query      fragment
                                                     (查询串)          (片段/锚点)
```

| 组件 | 说明 | HTTP 报文中的位置 |
| :--- | :--- | :--- |
| **scheme** | `http` / `https`（默认端口 80 / 443）；`ws` / `wss` 为 WebSocket 对应 | 不出现在请求行（选端口/协议用途） |
| **host** | 域名或 IP（IPv6 需方括号 `[::1]`），DNS 解析目标 | 由 `Host` 头部 + 连接目的 IP 体现 |
| **port** | 省略即用 scheme 默认端口 | 体现在 TCP 目的端口 |
| **path** | 层级路径，段间以 `/` 分隔，区分大小写；`origin-form` 必需 | **请求行**的 request-target |
| **query** | `?` 之后，`key=value&key2=value2`；顺序与重复键合法 | 与 path 一起写入请求行 |
| **fragment** | `#` 之后，**仅浏览器/客户端使用**（页内锚、路由、滚动定位） | **不发送给服务器**（服务器永远看不到 `#` 后内容） |

- **request-target 四种形式**：`origin-form`（`/path?q=1`，普通请求）、`absolute-form`（`http://host/p`，代理请求）、`authority-form`（`host:443`，`CONNECT`）、`asterisk-form`（`*`，`OPTIONS` 服务器级）。
- 路径规范化要防**目录穿越**（`/a/../../etc/passwd`）与 `//`、`%2e%2e` 变体；服务器应解码后校验再解析。

### 6.2 百分号编码（Percent-Encoding）

```text
安全集（unreserved）  A-Z a-z 0-9 - . _ ~            → 原样保留
其余一律              %HH（UTF-8 字节的十六进制，大写）
```

| 原文 | 编码 | 备注 |
| :--- | :--- | :--- |
| 空格 | path/query 中 `%20`；表单 `application/x-www-form-urlencoded` 中 `+` | 两者语义等价但写法不同，跨语言易错 |
| `&` `=` `?` `/` `#` `%` | `%26` `%3D` `%3F` `%2F` `%23` `%25` | **保留字符必须编码**，否则改变参数结构 |
| `中文` | `%E4%B8%AD%E6%96%87` | UTF-8 每汉字 3 字节 → 9 字符；URL 越长易被代理截断 |
| emoji | 4 字节 → `%F0%9F%98%80` | 代理对（surrogate pair）拼接错误会产生非法序列 |

> [!WARNING] 编码/解码的层次
> 只在**最终组件内**编码：先按结构拼出 path/query，再逐值编码（`URLSearchParams` / `encodeURIComponent`），**绝不对已编码串二次编码**（`%2526` 即双重编码 `&`）。签名/校验类协议必须固定"规范化 → 编码"顺序，否则同一请求产生两种签名。

---

## 🍪 7. 会话保持：Cookie 与 Session / Token

HTTP 无状态，多请求共享状态需在应用层构造。

### 7.1 Cookie 机制

```text
响应: Set-Cookie: sid=9f2c…; Domain=.example.com; Path=/api;
               Max-Age=1800; HttpOnly; Secure; SameSite=Lax
      ↓ 浏览器存入 Cookie Jar（按 domain+path 匹配）
后续请求: Cookie: sid=9f2c…      ← 浏览器自动附带，服务器据此恢复会话
```

| 属性 | 作用 | 注意 |
| :--- | :--- | :--- |
| **Domain** | 生效域（`.example.com` 含子域） | 只能设为当前主机本身或其父域（防投毒）；纯 IP 无法设域 cookie；未指定=仅精确主机 |
| **Path** | 生效路径前缀（`/api` 含 `/api/x`） | 更长的 path 优先匹配；不同 path 可放同名不同值 |
| **Expires / Max-Age** | 过期时间 / 相对秒数 | 二者同现时 `Max-Age` 优先；都不写 = **会话 cookie**（浏览器关闭即失效，非"会话结束"） |
| **HttpOnly** | 禁止 JS 通过 `document.cookie` 读取 | 显著降低 XSS 窃取会话的概率；对网络抓包无效 |
| **Secure** | 仅经 HTTPS 发送 | 现代浏览器要求 `SameSite=None` 必须同时带 `Secure` |
| **SameSite** | 跨站携带策略：`Strict`（任何跨站不带）/ `Lax`（默认，跨站顶层 GET 才带）/ `None`（全部携带，须 `Secure`） | 防 **CSRF** 的主手段；`None` 需显式声明，第三方场景（内嵌、回跳）才用 |
| **Partitioned** | 按"顶层站点"分区存储（CHIPS） | 第三方 cookie 限制下的嵌入式场景替代方案 |
| **Priority** | 存储配额紧张时的保留优先级（Low/Medium/High） | 与配额策略相关 |

限制与实践：单条 cookie ≈ 4 KB、每域条数与总量有上限（浏览器实现各异）；cookie 随**同域每个请求**（含图片/脚本）发送，占带宽且无法用 CORS 完全屏蔽；`Set-Cookie` 是唯一由服务器下发、其余同名 `Cookie` 由浏览器聚合的字段。

### 7.2 Session vs Token

| 维度 | Session + Cookie（服务端状态） | Token（JWT，无服务端状态） |
| :--- | :--- | :--- |
| 原理 | 服务器保存会话表，只把 **session id** 交给 cookie；请求时查表恢复 | JWT 一句话：**服务端用密钥对 `{claims}` 签名发出 `header.payload.signature` 三段 Base64URL 串，客户端每次带上（通常 `Authorization: Bearer`），服务端只验签不查库**，从而恢复身份 |
| 存储 | 服务端内存/Redis，水平扩容需共享存储或粘滞 | 客户端本地（前端 `localStorage`/内存），服务端无状态易扩容 |
| 注销 | 服务端删会话即刻失效 | 天然难注销，需黑名单/短有效期 + 刷新令牌（refresh token） |
| 篡改风险 | id 无信息，怕的是会话固定与 id 泄露 | payload **仅编码不加密**（Base64URL 可解码），绝不放敏感数据；怕算法降级（`alg:none`）与弱密钥 |
| 携带 | 浏览器自动（受 SameSite 影响，易被 CSRF） | 手动放头部，不受 CSRF 影响，但存 `localStorage` 会被 XSS 读走 |
| 典型组合 | 传统服务端渲染站点：`HttpOnly + SameSite=Lax` 的 session cookie | 前后端分离/App/API：JWT 放 `Authorization`，或"JWT 放 HttpOnly cookie + CSRF token 双提交" |

> [!TIP] 落地建议
> 会话类 cookie 一律 `HttpOnly; Secure; SameSite=Lax`；身份凭据有效期做短（分钟级）+ 刷新机制；跨站请求防护不要只依赖 Referer 判断；任何"记住我"类长有效期令牌都必须可单独吊销。

---

## 🔒 8. HTTPS：HTTP over TLS

HTTPS **不是新协议**，只是把 HTTP 报文放进 TLS 记录层再交给 TCP：`HTTP → TLS → TCP`（HTTP/3 则是 `HTTP/3 → QUIC → UDP`，加密在 QUIC 内完成）。

### 8.1 TLS 握手简述（以 TLS 1.3 一轮往返为主）

```text
客户端                                   服务器
  │  ClientHello ────────────────────────►│  支持的密码套件、密钥共享（key_share）、
  │   · SNI: www.example.com              │  ALPN: h2 / http/1.1
  │◄──────────────────────── ServerHello ──│  选定套件 + 服务器密钥共享
  │◄──────────── {证书链 / CertificateVerify / Finished}
  │  验证：链上签名 → 到期 → 主机名匹配 → 不受信任/撤销检查
  │◄──────────────── { 会话票据 NewSessionTicket }
  │  { Finished } ───────────────────────►│  此后即应用数据（HTTP 请求）
  │        握手 1-RTT；带 PSK/pre_shared_key 可 0-RTT
```

要点：**ECDHE 临时密钥 → 前向保密（PFS）**；**证书做身份认证而非密钥交换**（RSA 密钥交换已在 TLS 1.3 移除）；会话恢复靠 PSK/0-RTT（有重放风险，仅幂等请求可用）。TLS 1.2 需 2-RTT 完整握手、1-RTT 恢复。

> [!NOTE] 本笔记只讲 HTTP 与 TLS 的关系
> 完整握手细节、密码套件、证书与 PKI、TLS 1.2 全流程见 [[TLS]]。

### 8.2 HTTPS 对 HTTP 的影响

| 方面 | 说明 |
| :--- | :--- |
| 报文语义 | **完全不变**（方法/状态码/头部/正文同一套），只是被加密与防篡改（AEAD） |
| 明文暴露内容 | 加密后对链路观察者仍可见：IP/端口、SNI 中的域名（ECH 可隐藏）、证书主题、包长与时序（可推测站点）、DNS 查询 |
| HTTP 头部 | 头部**同样被加密**（TLS 记录层之内），只有 TCP/IP 层可见；`Host` 内容可被 SNI 关联 |
| 端口与 URL | 默认 443；`https://` scheme；代理需 `CONNECT` 才能透传 |
| ALPN 协商 | 握手中通过 ALPN 决定用 `h2`/`h3`/`http/1.1`——这就是"H2 部署几乎总在 HTTPS 上"的原因（浏览器实际只支持加密 H2/H3） |
| 混合内容 | HTTPS 页面加载 HTTP 子资源 = 混合内容，浏览器会拦截被动内容（图片）之外的主动内容（脚本/iframe） |
| HSTS | 响应头 `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload` 令浏览器此后强制走 HTTPS，避免 SSL 剥离 |
| 跳转与回退 | 常见 80 → 443 用 **301/308**；避免"先给 HTTP 页再跳"造成凭据外泄；重定向循环多为 LB 终结 TLS 而后端未识别 `X-Forwarded-Proto` |
| 性能成本 | 握手额外 RTT 与 CPU（可用会话票据、连接复用、TLS 1.3 0-RTT 摊薄）；**长连接收益在 HTTPS 下更大**（TLS 握手比 TCP 贵得多） |

---

## ⏱️ 9. 一次完整请求时序（浏览器 → https://www.example.com/）

```text
① DNS 解析    www.example.com → 93.184.216.34   （UDP 53，1 RTT；结果按 TTL 缓存）
② TCP 握手    SYN → SYN-ACK → ACK                （1 RTT，四元组确定）
③ TLS 握手    ClientHello → ServerHello/证书/Finished（TLS1.3 1 RTT；含 ALPN=h2）
④ 请求        GET / HTTP/1.1 / Host / User-Agent / Accept-Encoding / Cookie
⑤ 响应        HTTP/1.1 200 OK / Content-Type: text/html; charset=utf-8
              Cache-Control: no-cache / ETag: "a1b2" / 正文 HTML
⑥ 子资源      解析 HTML 得 CSS/JS/图片 → 复用连接（H1 keep-alive 或 H2 多流并发）
⑦ 后续请求    条件请求 If-None-Match → 304（只传头部，零正文）
⑧ 结束        keep-alive 空闲到超时 → FIN/ACK 挥手；H2 GOAWAY；H3 连接关闭帧
```

| 阶段 | 主要耗时来源 | 优化手段 |
| :--- | :--- | :--- |
| DNS | 解析链长、TTL 短 | 预解析 `dns-prefetch`、本地/EDNS 缓存、减少 CNAME 跳数 |
| TCP | 慢启动、窗口小 | 调初始拥塞窗口、连接复用、TCP Fast Open |
| TLS | 证书链与握手 RTT、CPU | TLS 1.3、会话票据/0-RTT、OCSP stapling、ECDSA 证书 |
| 请求-首字节 | 服务端处理、后端扇出 | 缓存（本地/CDN/Redis）、池化、超时链对齐 |
| 传输 | 带宽、跳数、头部冗余 | 压缩（gzip/br/zstd）、头部压缩（HPACK/QPACK）、`Content-Length` 正确 |
| 复用 | 连接池上限、空闲超时 | H1 keep-alive、H2/H3 多路复用、合理 `max_idle_timeout` |

> [!TIP] 冷启动 RTT 估算
> 首次访问一个 HTTPS 站点，**首字节至少 ≈ DNS(1) + TCP(1) + TLS(1~2) + 请求(1) = 4~5 个 RTT**；HTTP/3（QUIC）把 TCP+TLS 合并为 1 个 RTT，配合 0-RTT 恢复可到 1 RTT。这正是"连接复用/保活"价值远大于"报文本身大小"的原因。

---

## 🖼️ HTTP 报文结构与请求时序图

![[HTTP.excalidraw]]

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] HTTP 服务设计与联调必过
> - [ ] **报文语法**：起始行 + 头部 + **空行 CRLF** 完整；正文长度由 `Content-Length` 或 `chunked` 明确界定，禁止两者歧义同现（防请求走私）
> - [ ] **方法语义**：GET/HEAD 安全、PUT/DELETE 幂等；副作用一律放 POST/PATCH；非幂等操作配幂等键
> - [ ] **状态码正确**：401（未认证，带 `WWW-Authenticate`）≠ 403（无权限）；204/304 **无正文**；重定向按是否可改方法选 302/307、301/308
> - [ ] **必需头部**：HTTP/1.1 请求必须有 `Host`；跨域场景 `Origin` + `Vary: Origin` 一致；代理链重写 `Connection`/`X-Forwarded-*`
> - [ ] **URL 编解码**：逐值编码（`%20` vs `+` 统一）、杜绝双重编码；path 规范化防目录穿越；敏感参数不放 query（进日志/Referer）
> - [ ] **会话与安全**：cookie 齐备 `HttpOnly; Secure; SameSite=Lax`；JWT 只验签不存敏感数据，短有效期 + 可吊销；CSRF 有防护
> - [ ] **缓存策略**：静态指纹资源 `max-age=31536000, immutable`；入口页 `no-cache` + `ETag`；共享缓存看 `s-maxage`/`Vary`；正确产生 304
> - [ ] **HTTPS**：全站 HSTS（含 `preload` 前先确认无明文依赖）；证书链完整、主机名匹配、到期监控；80 强制 301/308 到 443；ALPN 与期望版本一致
> - [ ] **连接管理**：keep-alive 空闲超时有界（否则被 LB 先断导致竞态）；H2/H3 单连接多路复用，理解队头阻塞边界
> - [ ] **超时与重试**：客户端超时 < 网关超时链自洽；仅对幂等请求自动重试；429/503 遵守 `Retry-After` 并退避加抖动
> - [ ] **大小与限流**：限制头部总大小与正文大小（431/413）；慢客户端用背压/超时防 Slowloris（`408`）
> - [ ] **可观测**：记录方法、path（脱敏 query）、状态码、耗时、上游状态（502/504 能定位到具体上游）

---

## 🔗 思考与延伸索引

- [[TCP]] — HTTP/1.x 与 HTTP/2 的承载层：三次握手、四次挥手、滑动窗口与队头阻塞的根源
- [[UDP]] — HTTP/3 之下 QUIC 的承载层：丢包靠应用侧感知与重传
- [[IP]] — 网际层寻址与分片：MTU/PMTUD 对 QUIC 数据报大小的约束
- [[TLS]] — 证书链、密码套件与握手全流程（本笔记 §8 只讲与 HTTP 的接缝）
- [[SSH]] — 同为"应用层协议 + 传输层安全"的对照设计；端口转发可作 HTTP 调试隧道
- [[mqtt]] — 与 HTTP 互补的 IoT 传输：发布-订阅、长连接、有状态会话（对照 HTTP 的无状态请求-响应）
- [[DHCP]] — 客户端获取地址与 DNS 服务器，构成 §9 时序中 DNS 的前置条件
- [[FTP]] — 会话式文件传输协议对照：带内控制连接 + 数据连接，HTTP 用单连接 + 方法表达同类需求

---

## 📚 参考资料

1. **IETF** — *RFC 9110: HTTP Semantics*（方法、状态码、头部语义、缓存与条件请求）.
2. **IETF** — *RFC 9112: HTTP/1.1*（报文语法、持久连接、chunked 编码、消息路由）.
3. **IETF** — *RFC 9111: HTTP Caching*（`Cache-Control`/`Age`/陈旧应答规则）.
4. **IETF** — *RFC 9113: HTTP/2*（二进制分帧、流与多路复用、HPACK）.
5. **IETF** — *RFC 9114: HTTP/3* 与 *RFC 9000: QUIC*（QPACK、连接迁移、0-RTT）.
6. **IETF** — *RFC 3986: Uniform Resource Identifier (URI): Generic Syntax*；*RFC 6265: HTTP State Management Mechanism (Cookie)*；*RFC 6265bis* 草案（SameSite/Partitioned）.
7. **IETF** — *RFC 8446: The Transport Layer Security (TLS) Protocol Version 1.3*；*RFC 7301: ALPN*；*RFC 6797: HSTS*.
8. **MDN Web Docs** — *HTTP*（Overview / Requests / Responses / Status codes / Headers / Caching / CORS / Cookies）.
9. **RFC 9110 §9 / IANA** — *Hypertext Transfer Protocol (HTTP) Method Registry*、*Status Code Registry*、*Header Field Registry* 官方注册表.
10. **Mike Belshe 等** — *High Performance Browser Networking*（O'Reilly, 2013），HTTP/1.1→2 连接与传输成本分析.
