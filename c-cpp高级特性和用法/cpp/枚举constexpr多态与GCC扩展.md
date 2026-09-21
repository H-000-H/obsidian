---
title: C++ 作用域枚举、constexpr、多态与 GCC 扩展教学手册
tags:
  - CPP
  - Enum-Class
  - Constexpr
  - Polymorphism
  - GCC-Extensions
  - extern-C
  - Teaching
standard: C++17
status: Completed
created: 2026-09-20
---

# 🏷️ C++ 作用域枚举、constexpr、多态与 GCC 扩展教学手册

> [!INFO] 笔记概览
> - **核心命题**：现代 C++ 用 `enum class` 强类型、`constexpr` 编译期求值、`override/final` 安全多态替代旧式裸写法；GCC 扩展与内建函数则是嵌入式优化的利器。
> - **内容范围**：作用域枚举 · `constexpr` 常量/函数/`inline` 变量 · 纯虚多态 + `override`/`final` · `using` 别名 · 前向声明与匿名命名空间 · `__attribute__`/`__builtin_*` · `extern "C"` 混编
> - **推荐路径**：§1 enum class → §2 constexpr → §3 多态 → §4 using → §5 前向声明 → §6 GCC 扩展 → §7 extern "C"
> - **代码约定**：全部为**标准语法最小示例**；大括号遵循 Allman，trivial 单行存取器保持紧凑。

---

## 🏷️ 1. 作用域枚举 `enum class`

### 1.1 旧式 enum 的两个坑

```cpp
enum Color { red, green };      // 名字 red 污染全局作用域
enum Other  { red };            // 冲突！
int x = green;                  // 竟能隐式转成整数，类型不安全
```

### 1.2 enum class 修好这两点

```cpp
enum class Color  { red, green };
enum class Status { ok, error };

// Status::ok 不会和 Color::red 打架（作用域隔离）
// Color c = 0;                              // 错误：不能隐式转整数
int   x = static_cast<int>(Color::red);     // 拿整数要显式转换
Color c = static_cast<Color>(3);            // 整型回来也要显式
```

> [!TIP] 强类型 + 作用域
> `enum class` 不污染外层命名空间、不隐式转整数，是替代宏枚举的首选。可显式指定底层类型：`enum class Color : uint8_t { red, green };`，在嵌入式里节省空间。

---

## ⚡ 2. constexpr：编译期就算出来

### 2.1 constexpr 常量 / 函数

```cpp
constexpr int kMax = 100;                       // 编译期常量，能定数组大小
int arr[kMax];

constexpr int square(int x) { return x * x; }   // constexpr 函数
int v = square(5);                              // 编译期就算出 25
```

### 2.2 inline 变量：头文件里定义不报重复

C++17 前，`const`/`static` 在头文件里定义会"重复定义"报错；`inline` 允许头文件直接定义：

```cpp
// mycfg.hpp（被多个 .cpp include）
inline constexpr int kTimeoutMs = 1000;         // 放心写在头文件，不会重复定义
```

> [!NOTE] constexpr 的收益
> 编译期求值 → 少占可变 RAM、结果可进 Flash、启动即可用。常量优先 `constexpr` 而非宏（有类型检查）。`k` 前缀命名常量是本项目 C++ 约定（如 `kTimeoutMs`）。

---

## 🎭 3. 多态：纯虚函数 + override / final

### 3.1 纯虚函数定约定，子类实现

```cpp
#include <iostream>

class Shape
{
public:
    virtual ~Shape() = default;                 // 基类析构要 virtual
    virtual double area() const = 0;            // 纯虚：约定"都要能算面积"
};

class Circle : public Shape
{
public:
    double area() const override { return 3.14 * m_r * m_r; }   // override
private:
    double m_r = 1.0;
};

void print(const Shape& s) { std::cout << s.area(); }   // 传啥都按各自实现算
```

### 3.2 override 与 final

- `override`：声明"我在覆写基类函数"——签名写错编译器立刻报错，而不是悄悄新开一个函数。
- `final`：声明"到此不能再被覆写/继承"。

```cpp
class Special final : public Circle { };        // final：不能再被继承
```

> [!CAUTION] 虚函数的开销
> 虚函数带来 vtable（占 Flash/RAM）与运行期间接跳转。嵌入式热路径优先**静态分派**（CRTP / 模板 / 函数表），仅在确需运行期多态时用虚函数。静态多态见 [[模板与泛型编程]] §4。

---

## 🔤 4. using 类型别名

```cpp
using Callback = void (*)(int, void*);                   // 函数指针别名
using MyVector = std::vector<int>;                       // 容器别名
template <typename T> using OwnPtr = std::unique_ptr<T>; // 模板别名（typedef 做不到）
```

> [!TIP] using 优于 typedef
> `using` 支持模板别名、可读性更好（`using X = ...` 比 `typedef ... X` 直观）。新代码统一用 `using`。

---

## 🙈 5. 前向声明与匿名命名空间

### 5.1 前向声明：只用"类型存在"，不必 include

```cpp
struct Device;                          // 只声明，别 include 大头部文件
class Led { Device* m_dev; };           // 用指针/引用即可，无需完整定义
```

### 5.2 匿名命名空间：仅本文件可见

```cpp
namespace {                             // 仅本翻译单元可见（等价于内部链接）
    const int kFactor = 3;
    void helper() {}
}
```

> [!NOTE] 减少编译依赖
> 前向声明能显著降低头文件耦合、加快编译；匿名命名空间把实现细节限制在单个 `.cpp` 内，避免符号污染与 ODR 冲突。

---

## 🛠️ 6. GCC 扩展与内建函数

GCC / Clang 可用以下语法辅助嵌入式优化（非标准，但主流工具链都支持）。

### 6.1 属性 `__attribute__`

```cpp
__attribute__((always_inline)) int fast(void);    // 强制内联
struct Foo { } __attribute__((aligned(4)));        // 结构体 4 字节对齐
__attribute__((used)) int entry(void);             // 强制保留，避免被优化丢弃
```

### 6.2 内建位运算：`__builtin_ctz` / `__builtin_popcount`

```cpp
__builtin_ctz(0b1000);          // 尾随 0 的个数 = 3（最低置位位的位置）
__builtin_popcount(0b1101);     // 置位个数 = 3

// 位图分配最小可用位置，常用套路
int nextFree(unsigned used)
{
    return __builtin_ctz(~used) + 1;
}
```

> [!TIP] 内建函数的价值
> `__builtin_ctz` 直接映射到 CPU 的 CLZ/CTZ 指令（O(1)），比循环找位快得多。位图 + CTZ 的调度器实现见 [[嵌入式内核]]。

---

## 🔗 7. C/C++ 混编：`extern "C"`

C 和 C++ 的**名字修饰（name mangling）规则不同**。把 C 符号用 `extern "C"` 包起来，即可避免链接时符号对不上：

```cpp
extern "C" void system_init();          // C++ 里声明：按 C 链接规则查找符号

#ifdef __cplusplus
extern "C" {
#endif

void c_impl(void);                      // 头文件里同时被 C / C++ include 的安全写法

#ifdef __cplusplus
}
#endif
```

> [!NOTE] 何时需要
> C++ 调用 C 库、或 C 调用 C++ 导出的接口时，必须用 `extern "C"` 关闭名字修饰。注意 `extern "C"` 函数**不能重载**（C 无重载语义）。C 侧的设备模型与 OOP 技巧见 [[面向对象C与设备模型]]。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 写现代 C++ 前逐条核对
> - [ ] **枚举**：用 `enum class` 替代裸 `enum`/宏，必要时指定底层类型
> - [ ] **常量**：优先 `constexpr` + `k` 前缀命名，而非宏
> - [ ] **头文件常量**：C++17 用 `inline constexpr` 避免重复定义
> - [ ] **多态**：覆写必标 `override`，终结类标 `final`，基类析构 `virtual`
> - [ ] **虚函数克制**：热路径优先静态分派（CRTP/模板/函数表）
> - [ ] **别名**：用 `using` 替代 `typedef`，支持模板别名
> - [ ] **前向声明**：能用前向声明就别 include，降低耦合
> - [ ] **匿名命名空间**：实现细节放匿名 namespace，限内部链接
> - [ ] **混编**：C/C++ 接口用 `extern "C"`，注意不能重载

---

## 🔗 思考与延伸索引

- [[模板与泛型编程]] — CRTP 静态多态，替代虚函数的另一条路
- [[类型擦除与轻量 RTTI]] — 运行期多态与 `static_assert` 编译期约束
- [[Lambda 回调与单例模式]] — `using` 回调别名、`explicit`/`=delete` 的实战
- [[ETL容器与萃取]] — `constexpr` 定容容器与 `type_traits` 萃取
- [[嵌入式内核]] — `__builtin_ctz` 位图调度、`__attribute__` 内核技巧
- [[C语言高级用法]] — C 侧的 `extern "C"`、指定初始化器等对应技巧

---

## 📚 参考资料

1. **cppreference.com** — *Scoped enumerations* / *constexpr* / *override* / *using-declaration*.
2. **GCC Manual** — *Attribute Syntax* / *Other Built-in Functions*（`__builtin_ctz` 等）.
3. **Scott Meyers** — *Effective Modern C++*（Item 10：`enum class`；Item 15：`constexpr`）.
4. **ISO/IEC 14882:2017** — *C++17 Standard*（`inline` 变量、`enum class`、`extern "C"`）.
