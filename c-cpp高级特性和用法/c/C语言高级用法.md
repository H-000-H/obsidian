---
title: C 语言高级用法基础教学手册（指定初始化器 / 数组长度宏 / 查表 / 断言宏 / extern "C"）
tags:
  - C
  - Designated-Initializer
  - Macro
  - Lookup-Table
  - Assert
  - extern-C
  - Teaching
standard: C99/C11
status: Completed
created: 2026-09-20
---

# 📘 C 语言高级用法基础教学手册

> [!INFO] 笔记概览
> - **核心命题**：这批是 C 里"基础但天天用"的技巧——指定初始化器抗错位、数组长度宏免手算、`static const` 查表进只读段、断言宏定位到行、`extern "C"` 混编。
> - **内容范围**：指定初始化器 · `sizeof` 算数组长度 · `static const` 结构体查表 · 断言宏（`__FILE__`/`__LINE__`）· C/C++ 混编 `extern "C"`
> - **推荐路径**：§1 指定初始化器 → §2 数组长度 → §3 查表 → §4 断言宏 → §5 extern "C"
> - **代码约定**：最原生 C 语法；**Allman 大括号、4 空格缩进、控制语句强制花括号**。

---

## 🏷️ 1. 指定初始化器（Designated Initializers）

C99：初始化结构体/数组时**按成员或下标名赋值**，而不是按位置挨个填。

```c
struct Point
{
    int x;
    int y;
};

struct Point p1 = { 10, 20 };            /* 位置初始化：要记得顺序 */
struct Point p2 = { .y = 20, .x = 10 };  /* 指定初始化：顺序随意，可读性强 */

/* 只填关心的字段，其余自动 0 */
struct Device
{
    int  id;
    int  port;
    long address;
};
struct Device d = { .address = 0x08000000 };
```

> [!TIP] 三大好处
> 抗结构调整（`Point` 加个字段不会错位）、只写关心的字段（其余自动零初始化）、自带可读性（字段名即注释）。驱动填 `file_operations` 表全靠它，见 [[面向对象C与设备模型]] §1。

---

## 📏 2. 编译期算数组长度：`sizeof(a) / sizeof(a[0])`

```c
int arr[] = { 5, 1, 4 };
size_t n = sizeof(arr) / sizeof(arr[0]);   /* 12 / 4 = 3 */

#define ARRAY_LEN(a) (sizeof(a) / sizeof((a)[0]))   /* 建议整体加括号，元素也加括号 */

uint8_t table[] = { /* ... */ };
size_t count = ARRAY_LEN(table);           /* 加删元素不怕，自动算 */
```

> [!CAUTION] 只对数组有效
> `ARRAY_LEN` 对**真正的数组**有效；对退化成指针的函数参数（`uint8_t *a`）会得到错误结果（指针大小 / 元素大小）。传数组进函数时务必同时传长度。

---

## 📋 3. `static const` 结构体查表

把"只读配置/几何数据"预置成静态常量结构体数组，运行只查不改、放只读段：

```c
typedef struct
{
    uint32_t offset;
    uint32_t size;
} sec_info_t;

static const sec_info_t layout[] =
{
    { 0x08000000, 0x04000 },   /* 扇区 0 */
    { 0x08010000, 0x10000 },   /* 扇区 1 */
};

for (size_t i = 0; i < ARRAY_LEN(layout); i++)
{
    if (layout[i].offset == target)
    {
        /* 命中 */
    }
}
```

> [!NOTE] 三词配合
> `static`（内部链接，不泄漏出文件）+ `const`（进只读段/Flash）+ 结构体数组（自描述表）三词配合很常见。固定表用 `const` 数组，数据进 Flash、无堆、启动即可用。

---

## 🛑 4. 断言宏：`__FILE__` / `__LINE__`

宏能拿到编译器预定义的 `__FILE__`/`__LINE__`，表达式为假就定位到文件行号：

```c
#define ASSERT(expr) \
    ((expr) ? (void)0 : assert_failed((char *)__FILE__, __LINE__))

void assert_failed(char *file, unsigned line)
{
    printf("断言失败 %s:%u\n", file, line);
    while (1)
    {
        /* 停在现场调试 */
    }
}
```

可带编译开关，生产版零开销：

```c
#ifdef USE_ASSERT
#define ASSERT(e) ((e) ? (void)0 : assert_failed(__FILE__, __LINE__))
#else
#define ASSERT(e) ((void)0)
#endif
```

> [!TIP] 为什么必须用宏
> 用**宏**才能拿到调用点的 `__FILE__`/`__LINE__`（函数做不到——函数里取到的是函数自己的位置）；`printf` 也可配合 `#e`（字符串化）打印出表达式文本，便于定位。

---

## 🔗 5. C 与 C++ 混编：`extern "C"`

C++ 为支持重载会把函数名 mangle（名字修饰），C 不 mangle，两边符号对不上就链接失败。用 `extern "C"` 包住 C 声明：

```cpp
#ifdef __cplusplus
extern "C" {        /* 只对 C++ 编译器生效 */
#endif

void flash_init(void);    /* C 语言，按 C 的链接规则 */

#ifdef __cplusplus
}
#endif
```

> [!NOTE] 链接规则对齐
> 于是 C++ 侧 `flash_init()` 会按 C 的链接规则去找符号，链接对上。注意 `extern "C"` 函数**不能重载**。C++ 侧的对应说明见 [[枚举constexpr多态与GCC扩展]] §7。

---

## 📊 6. 小结

| 特性 | 一句话 |
| :--- | :--- |
| 指定初始化器 | 按成员名初始化，抗错位、可读 |
| `sizeof/元素` 表长宏 | 数组长度自动算，改表不怕 |
| `static const` 结构体表 | 只读查表，占只读段 |
| 断言宏 + `__FILE__`/`__LINE__` | 失败定位到文件行，可编译时关 |
| `extern "C"` | C++ 无损调用 C 函数 |

---

## 🔗 思考与延伸索引

- [[枚举constexpr多态与GCC扩展]] — `extern "C"` 的 C++ 侧对应、`enum class`、`constexpr`
- [[C语言的宏魔法]] — 断言宏之外的 `do-while(0)`、`#error`、`_Static_assert` 等宏技巧
- [[面向对象C与设备模型]] — 指定初始化器填 vtable、静态设备表查表
- [[嵌入式内核]] — `static const` 表、`_Static_assert` 在内核里的用法

---

## 📚 参考资料

1. **ISO/IEC 9899:1999/2011** — *C99/C11 Standard*（指定初始化器、`offsetof`、`_Static_assert`）.
2. **cppreference.com** — *Initialization* / *extern "C"*（C/C++ 混编）.
3. **《C 和指针》/《C 专家编程》** — 数组与指针退化、断言、链接属性.
4. 项目内部笔记：[[C语言的宏魔法]]、[[面向对象C与设备模型]]。
