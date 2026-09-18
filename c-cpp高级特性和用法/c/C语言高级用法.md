# C 语言高级用法基础

用最原生 C 语法讲这批基础但常用的技巧。

## 一、指定初始化器（Designated Initializers）

C99：初始化结构体/数组时**按成员或下标名赋值**，而不是按位置挨个填。

```c
struct Point { int x, y; };

struct Point p1 = { 10, 20 };          // 位置初始化：要记得顺序
struct Point p2 = { .y = 20, .x = 10 }; // 指定初始化：顺序随意, 可读性强

// 只填关心的字段, 其余自动 0
struct Device { int id; int port; long address; };
struct Device d = { .address = 0x08000000 };
```

好处：抗结构调整（`Point` 加个字段不会错位）、只写关心的、自带可读性。

## 二、编译期算数组长度：`sizeof(a) / sizeof(a[0])`

```c
int arr[] = { 5, 1, 4 };
size_t n = sizeof(arr) / sizeof(arr[0]);   // 12 / 4 = 3

#define ARRAY_LEN(a) (sizeof(a) / sizeof((a)[0]))   // 建议 package 括号
uint8_t table[] = { ... };
size_t count = ARRAY_LEN(table);           // 加删元素不怕, 自动算
```

## 三、`static const` 结构体查表

把"只读配置/几何数据"预置成静态常量结构体数组，运行只查不改、放只读段：

```c
typedef struct { uint32_t offset; uint32_t size; } sec_info_t;
static const sec_info_t layout[] = {
    { 0x08000000, 0x04000 },   // 扇区 0
    { 0x08010000, 0x10000 },   // 扇区 1
};
for (size_t i = 0; i < ARRAY_LEN(layout); i++)
    if (layout[i].offset == target) { /* 命中 */ }
```

`static`（内部链接，不泄漏出文件）+ `const`（进只读段）+ 结构体数组（自描述表）三词配合很常见。

## 四、断言宏：`__FILE__` / `__LINE__`

宏能拿到编译器预定义的 `__FILE__`/`__LINE__`，表达式为假就定位到文件行号：

```c
#define ASSERT(expr) \
    ((expr) ? (void)0 : assert_failed((char*)__FILE__, __LINE__))

void assert_failed(char* file, unsigned line)
{
    printf("断言失败 %s:%u\n", file, line);
    while (1) { }    // 停在现场调试
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

要点：用**宏**才能拿到 `__FILE__`/`__LINE__`（函数做不到）；`__print` 也可配合 `#e` 打印表达式。

## 五、C 与 C++ 混编：`extern "C"`

C++ 为支持重载会把函数名 mangle，C 不 mangle，两遍符号对不上就链接失败。用 `extern "C"` 包住 C 声明：

```cpp
#ifdef __cplusplus
extern "C" {        // 只对 C++ 编译器生效
#endif

void flash_init(void);    // C 语言, 按 C 的链接规则

#ifdef __cplusplus
}
#endif
```

于是 C++ 侧 `flash_init()` 会按 C 的链接规则去找符号，链接对上。

## 小结

| 特性 | 一句话 |
| --- | --- |
| 指定初始化器 | 按成员名初始化，抗错位、可读 |
| `sizeof/元素` 表长宏 | 数组长度自动算，改表不怕 |
| `static const` 结构体表 | 只读查表，占只读段 |
| 断言宏 + `__FILE__`/`__LINE__` | 失败定位到文件行，可编译时关 |
| `extern "C"` | C++ 无损调用 C 函数 |

## 关联
- [[枚举constexpr多态与GCC扩展]]（extern "C" 的 C++ 侧重合）