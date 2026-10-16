---
title: C 语言的宏魔法与编译期技巧教学手册（do-while(0) / ##__VA_ARGS__ / #error / _Static_assert / __LINE__ 协程 / __atomic）
tags:
  - C
  - Macro
  - Preprocessor
  - Compile-Time
  - GCC-Extensions
  - Embedded
  - Teaching
standard: C11 + GNU 扩展
status: Completed
---

# 🪄 C 语言的宏魔法与编译期技巧教学手册

> [!INFO] 笔记概览
> - **核心命题**：C 的宏与预处理器能在**编译期**做大量工作——打包语句、生成代码、锁定布局、拒绝非法配置、甚至生成协程状态机，把运行期开销前移到编译期。
> - **内容范围**：`do-while(0)` · `##__VA_ARGS__` · `#`/`##` · `#error` · `_Static_assert`+`offsetof` · `__has_include` · `packed`/`aligned` · `__LINE__` 协程 · `__builtin_*`/`__atomic_*`
> - **推荐路径**：§1 do-while(0) → §2 变参宏 → §3 字符串化/连接 → §4 #error → §5 静态断言 → §6~7 属性 → §8 协程 → §9 原子
> - **代码约定**：最原生 C（标准 C + GNU 扩展），去掉工程自定义宏名；**Allman 大括号、4 空格缩进、控制语句强制花括号**。

---

## 🔁 1. `do { ... } while (0)`：把多句宏包成"一个语句"

多语句宏直接展开会在 `if/else` 下出错，用 `do-while(0)` 包住：

```c
#define PRETTY_LOG(fmt, ...)                                       \
    do {                                                           \
        time_t ts = get_tick();                                    \
        printf("[%lu] %s " fmt "\n",                               \
               (unsigned long)ts, __func__, ##__VA_ARGS__);        \
    } while (0)
```

> [!NOTE] 为什么不用裸 `{...}`
> 若写成裸 `{...}`，`if (x) PRETTY_LOG(...); else ...` 在 `if` 分支里会多出个分号导致 `else` 悬空。`do-while(0)` 无循环开销，展开后就是单语句 + 末尾分号，能安全套进任何 `if/else`。

---

## 📎 2. `##__VA_ARGS__`：可变参数宏的 GNU 扩展

```c
#define LOG_I(fmt, ...) log_emit("[INFO]", fmt, ##__VA_ARGS__)
```

- `...` 捕获变参，`fmt` 后面的内容原样透传。
- `##` 让**没有变参时自动删掉前面的逗号**，从而 `LOG_I("你好")` 也能编译（不会留下悬空逗号）。

---

## 🔗 3. `#` 字符串化 与 `##` 连接

```c
#define STR(x)   #x        /* # 把形参变成字符串字面量 */
#define CAT(a,b) a##b       /* ## 把两个 token 拼成一个 */

STR(hello)            /* → "hello" */
int CAT(a, _count);   /* → 声明成 int a_count; */
```

> [!TIP] 代码生成两大利器
> `##` 常用于宏代码生成（自动命名函数/变量），`#` 常用于打印值或做编译期消息。设备驱动注册的 `##` 拼接见 [[面向对象C与设备模型]] §4。

---

## ⛔ 4. `#error`：编译期直接拒绝非法配置

```c
#if (RING_SIZE & (RING_SIZE - 1)) != 0
#error "RING_SIZE must be a power of two"
#endif
```

> [!NOTE] 2 的幂检查
> `x & (x-1) == 0` 是"判断 2 的幂"的经典写法（只有单个置位时成立）。配错编译即报，不用等运行崩。

---

## 🔒 5. `_Static_assert` / `offsetof`：编译期断言 + 锁结构布局

```c
#include <stddef.h>

_Static_assert(N >= 1, "N must be >= 1");       /* 编译期断言，C11 */

typedef struct
{
    uint32_t sp;
    uint32_t pc;
} vtable_min_t;

_Static_assert(offsetof(vtable_min_t, pc) == 4u,
               "pc 必须在偏移 4（ARM 向量表布局）");
```

> [!TIP] 精度敏感场景必备
> `offsetof` 取成员偏移，配静态断言可在**编译期锁定结构体内存布局**（如协议头、向量表、DMA 描述符这种精度敏感场景），布局一旦错位编译即失败。

---

## 📂 6. `__has_include`：按"能否包含"选配置

```c
#if defined(__has_include)
  #if __has_include("config.h")
    #include "config.h"    /* 用户放一个 config.h 覆盖默认 */
  #endif
#endif
```

让用户"可选地自定义"，无则用默认，提升可移植性。

---

## 📐 7. `__attribute__((packed))` / `aligned`

```c
/* 去填充，用于网络帧/日志帧 */
struct frame_hdr
{
    uint16_t len;
    uint16_t crc;
} __attribute__((packed));

#define ALIGN_TO(x) __attribute__((aligned(x)))

ALIGN_TO(64) uint16_t w_ptr;   /* 对齐 64B，独占缓存行，减核间/中断争用 */
```

> [!CAUTION] packed 的代价
> `packed` 去字节填充但可能导致**非对齐访问**（某些架构会触发 HardFault 或降速）；`aligned(64)` 对齐缓存行，是做并发/缓存优化的细节。Cache 行与多核争用见 [[SPI-DMA-多核-Cache]]。

---

## 🧵 8. `__LINE__` 拼接：宏生成协程状态机（protothread）

单栈协程用 `switch-case` 靠 `__LINE__` 自动编号，免手工状态号：

```c
#define PT_BEGIN(t)   switch ((t)->pt_line) { case 0:
#define PT_YIELD(t)   (t)->pt_line = __LINE__;                    \
                      __attribute__((fallthrough)); case __LINE__:
#define PT_END(t)     }

/* 用法：在函数里像顺序代码一样写，遇到 PT_YIELD 自动记住"返回到这行继续" */
int task_loop(task_t *t)
{
    PT_BEGIN(t);
    step_a();
    PT_YIELD(t);        /* 暂停并记住 __LINE__ */
    step_b();
    PT_YIELD(t);
    /* ... */
    PT_END(t);
    return 0;
}
```

> [!NOTE] 精华
> `__LINE__` 编译期是唯一整数，等于自动发状态号；`.c` 改了行号也自动对得上。这是"宏当状态机生成器"的进阶玩法（protothread / Duff's device 思想）。

---

## ⚛️ 9. 内建函数与原子内置：`__builtin_` / `__atomic_`

```c
__builtin_memcpy(dst, src, n);              /* 指定内建，易优化为单指令拷贝 */
__atomic_load_n (&q, __ATOMIC_ACQUIRE);     /* 无锁读 */
__atomic_store_n(&q, v, __ATOMIC_RELEASE);  /* 无锁写 */
```

> [!TIP] 无锁 SPSC
> 用 acquire/release 内存序实现**无锁单生产者单消费者（SPSC）**：生产端 release 写、消费端 acquire 读，跨线程/中断安全。环形缓冲的 SPSC 应用见 [[DMA、Buffer 与流量控制]]、[[UART-code]] §3。

---

## 📊 10. 小结

| 宏/技巧 | 一句话 |
| :--- | :--- |
| `do{...}while(0)` | 多句宏打包成单语句，安全套 `if` |
| `##__VA_ARGS__` | 变参可空，兼容无参调用 |
| `#` 字符串化 / `##` 连接 | 生成字符串 / 拼 token |
| `#error` + 2 的幂检查 | 非法配置编译即报 |
| `_Static_assert` + `offsetof` | 编译期锁结构布局 |
| `__has_include` | 可选配置头，可移植 |
| `packed` / `aligned` | 去填充 / 对齐缓存行 |
| `__LINE__` 拼接 | 宏生成协程状态机 |
| `__builtin_*` / `__atomic_*` | 指定内建、无锁并发基元 |

---

## 🔗 思考与延伸索引

- [[面向对象C与设备模型]] — `##` 代码生成、`constructor` 注册宏
- [[嵌入式内核]] — `_Static_assert`、volatile 寄存器宏、`__builtin_ctz`
- [[C语言高级用法]] — 指定初始化器、数组长度宏、断言宏基础
- [[DMA、Buffer 与流量控制]] — `__atomic_*` 无锁 SPSC 环形缓冲的应用场景
- [[SPI-DMA-多核-Cache]] — `aligned(64)` 缓存行对齐与多核伪共享

---

## 📚 参考资料

1. **GCC Manual** — *Macro Arguments* / *Variadic Macros* / *Attribute Syntax* / *Statement Exprs*.
2. **ISO/IEC 9899:2011** — *C11 Standard*（`_Static_assert`、`offsetof`、`__has_include` 为 C2x）.
3. **Adam Dunkels** — *Protothreads*（`__LINE__` 协程状态机原型）.
4. 项目内部笔记：[[嵌入式内核]]、[[面向对象C与设备模型]]。
