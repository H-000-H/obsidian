# C 语言的宏魔法与编译期技巧

用最原生 C 语法讲这些宏/编译期手法，去掉工程自定义宏名。

## 1. `do { ... } while (0)`：把多句宏包成"一个语句"

多语句宏直接展开会在 `if/else` 下出错，用 `do-while(0)` 包住：

```c
#define PRETTY_LOG(fmt, ...) \
    do { \
        time_t ts = get_tick(); \
        printf("[%lu] %s " fmt "\n", (unsigned long)ts, __func__, ##__VA_ARGS__); \
    } while (0)
```

**为什么**：若写成裸 `{...}`，`if (x) PRETTY_LOG(...); else ...` 在 `if` 分支里会多出个分号；`do-while(0)` 无循环开销，展开后就是单语句 + 末尾分号。

## 2. `##__VA_ARGS__`：可变参数宏的 GNU 扩展

```c
#define LOG_I(fmt, ...) log_emit("[INFO]", fmt, ##__VA_ARGS__)
```

- `...` 捕获变参，`fmt` 后面的内容原样透传。
- `##` 让**没有变参时自动删掉前面的逗号**，从而 `LOG_I("你好")` 也能编译。

## 3. `#` 字符串化 与 `##` 连接

```c
#define STR(x)  #x     /* # 把形参变成字符串字面量 */
#define CAT(a,b) a##b /* ## 把两个 token 拼成一个 */

STR(hello)          // "hello"
int CAT(a, _count); // 声明成 int a_count;
```

`##` 常用于宏代码生成（自动命名函数/变量），`#` 常用于打印值或做编译期消息。

## 4. `#error`：编译期直接拒绝非法配置

```c
#if (RING_SIZE & (RING_SIZE - 1)) != 0
#error "RING_SIZE must be a power of two"
#endif
```

`x & (x-1) == 0` 是"判断 2 的幂"的经典写法；配错编译即报，不用等运行崩。

## 5. `_Static_assert` / `offsetof`：编译期断言 + 锁结构布局

```c
#include <stddef.h>
_Static_assert(N >= 1, "N must be >= 1");       // 编译期断言, C11

typedef struct { uint32_t sp; uint32_t pc; } vtable_min_t;
_Static_assert(offsetof(vtable_min_t, pc) == 4u,
               "pc 必须在偏移 4(ARM 向量表布局)");
```

`offsetof` 取成员偏移，配静态断言可在**编译期锁定结构体内存布局**（如协议头、向量表这种精度敏感场景）。

## 6. `__has_include`：按"能否包含"选配置

```c
#if defined(__has_include)
  #if __has_include("config.h")
    #include "config.h"    // 用户放一个 config.h 覆盖默认
  #endif
#endif
```

让用户"可选地自定义"，无则用默认。

## 7. `__attribute__((packed))` / `aligned`

```c
struct frame_hdr { uint16_t len; uint16_t crc; } __attribute__((packed));  // 去填充, 网络帧/日志帧
#define ALIGN_TO(x) __attribute__((aligned(x)))
ALIGN_TO(64) uint16_t w_ptr;   // 对齐 64B, 独占缓存行, 减核间/中断争用
```

`packed` 去字节填充；`aligned(64)` 对齐缓存行，是做并发/缓存优化的细节。

## 8. `__LINE__` 拼接：宏生成协程状态机（protothread）

单栈协程用 `switch-case` 靠 `__LINE__` 自动编号，免手工状态号：

```c
#define PT_BEGIN(t)   switch ((t)->pt_line) { case 0:
#define PT_YIELD(t)   (t)->pt_line = __LINE__;       \
                      __attribute__((fallthrough)); case __LINE__:

// 用法: 在函数里像顺序代码一样写, 遇到 PT_YIELD 自动记住"返回到这行继续"
int task_loop(task_t* t)
{
    PT_BEGIN(t);
    step_a();
    PT_YIELD(t);        // 暂停并记住 __LINE__
    step_b();
    PT_YIELD(t);
    /* ... */
    PT_END(t);
}
```

**精华**：`__LINE__` 编译期是唯一整数，等于自动发状态号；`.c` 改了行号也自动对得上。这是"宏当状态机生成器"的进阶玩法。

## 9. 内建函数与原子内置：`__builtin_` / `__atomic_`

```c
__builtin_memcpy(dst, src, n);                    // 指定内建, 易优化为单指令拷贝
__atomic_load_n (&q, __ATOMIC_ACQUIRE);           // 无锁读
__atomic_store_n(&q, v, __ATOMIC_RELEASE);        // 无锁写
```

用 acquire/release 内存序实现**无锁单生产者单消费者（SPSC）**：生产端 release 写、消费端 acquire 读，跨线程/中断安全。

## 小结

| 宏/技巧 | 一句话 |
| --- | --- |
| `do{...}while(0)` | 多句宏打包成单语句，安全套 `if` |
| `##__VA_ARGS__` | 变参可空，兼容无参调用 |
| `#` 字符串化 / `##` 连接 | 生成字符串 / 拼 token |
| `#error` + 2 的幂检查 | 非法配置编译即报 |
| `_Static_assert` + `offsetof` | 编译期锁结构布局 |
| `__has_include` | 可选配置头，可移植 |
| `packed` / `aligned` | 去填充 / 对齐缓存行 |
| `__LINE__` 拼接 | 宏生成协程状态机 |
| `__builtin_*` / `__atomic_*` | 指定内建、无锁并发基元 |

## 关联
- [[面向对象 C 与设备模型]]（`##` 代码生成、constructor）
- [[嵌入式内核]]（_Static_assert、寄存器宏）