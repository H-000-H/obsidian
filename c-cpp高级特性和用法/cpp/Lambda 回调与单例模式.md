---
title: C++ Lambda、回调与单例模式教学手册（含 =delete / =default / explicit）
tags:
  - CPP
  - Lambda
  - Callback
  - Singleton
  - Design-Pattern
  - Teaching
standard: C++17
status: Completed
created: 2026-09-20
---

# 🎣 C++ Lambda、回调与单例模式教学手册

> [!INFO] 笔记概览
> - **核心命题**：Lambda 是"随手写的函数对象"，回调是"把做什么交给调用方"，单例是"整个程序只有一个实例"——三者常组合出现（如单例注册 lambda 回调）。
> - **内容范围**：Lambda 基本形态与捕获 · `mutable` · 回调三种写法（函数指针/`std::function`/模板）· 单例（函数内 static / CRTP 模板化）· `=delete` / `=default` / `explicit`
> - **推荐路径**：§1 Lambda → §2 回调 → §3 单例 → §4 特殊成员函数控制
> - **代码约定**：全部为**标准 C++ 原生写法**最小示例；大括号遵循 Allman，trivial 单行存取器与 lambda 保持紧凑。

---

## 🐑 1. Lambda：随手写个"函数对象"

### 1.1 基本形态

```cpp
auto f = [](int a, int b) { return a + b; };   // [捕获](参数) mutable -> 返回 { 函数体 }
int r = f(1, 2);                               // 3
```

### 1.2 捕获：lambda 能不能用外部变量

```cpp
int base = 10;
auto add = [base](int x) { return x + base; };   // 拷贝捕获：外部 base 只有只读副本

int counter = 0;
auto inc = [&counter]() { ++counter; };          // 引用捕获：能改外部 counter
```

| 捕获写法 | 含义 |
| :---: | :--- |
| `[x]` | 拷贝捕获 x（只读副本） |
| `[&x]` | 引用捕获 x（可改外部） |
| `[=]` | 所有外部变量按拷贝带入 |
| `[&]` | 所有外部变量按引用带入 |

### 1.3 易错点：按值捕获的变量默认不可改

```cpp
int n = 0;
auto a = [n]()          { n++; };   // 报错：n 是只读副本
auto b = [n]() mutable  { n++; };   // 加 mutable 才允许改这份副本（不影响外部 n）
```

> [!CAUTION] 捕获列表要短
> 嵌入式/业务代码里 lambda 捕获列表应尽量短，避免 `[=]`/`[&]` 一把梭把大对象或悬垂引用带进来。ISR 里只用 lambda 置位/入队，重逻辑放下半部。

---

## 🔔 2. 回调：把"做什么"交给调用方

把一个函数当作参数传进另一个函数，让对方在合适时机调它，常用于"事件发生后通知我"。三种写法各有取舍：

### 2.1 写法 A：函数指针（C 风格，零开销）

```cpp
void onPress() { std::cout << "按下\n"; }

void registerHandler(void (*cb)())
{
    cb();                              // 存起来按需调用
}

registerHandler(onPress);
```

### 2.2 写法 B：`std::function`（类型擦除，能装 lambda/成员函数）

```cpp
#include <functional>

std::function<void()> onEvent;
onEvent = []() { std::cout << "事件\n"; };
onEvent();                             // 统一调用
```

### 2.3 写法 C：模板 + 万能引用（零开销、支持完美转发）

```cpp
#include <utility>

template <typename Callback>
void registerHandler(Callback&& cb)
{
    auto f = std::forward<Callback>(cb);   // 原样保留值类别，不多拷贝
    f();
}

registerHandler([]{ std::cout << "hi\n"; });
```

> [!TIP] 三种写法怎么选
> **函数指针**：简单回调、C 接口、无捕获 lambda，零开销首选。**`std::function`**：需要存异构回调、可能堆分配（嵌入式慎用）。**模板**：追求零开销 + 完美转发，但每种回调类型都会实例化一份。类型擦除原理见 [[类型擦除与轻量 RTTI]]。

---

## 1️⃣ 3. 单例模式：整个程序只有一个实例

### 3.1 最简洁写法：函数内 `static` 局部对象

```cpp
#include <string>

class Logger
{
public:
    static Logger& getInstance()
    {
        static Logger instance;            // static 局部对象：只构造一次
        return instance;
    }
    void log(const std::string& s) { /* ... */ }

private:
    Logger() {}                            // 构造私有：别人 new 不了
    Logger(const Logger&) = delete;        // 拷贝禁止
    Logger& operator=(const Logger&) = delete;
};

Logger::getInstance().log("hello");        // 每次拿的都是同一个
```

> [!NOTE] 为什么这样最稳
> C++11 起，函数内 `static` 局部变量的初始化**自带线程安全**（Magic Static），不用自己加锁；构造私有 + 拷贝删除封死了创建副本的途径。

### 3.2 模板化（CRTP）：让每个子类自动单例

```cpp
template <typename T>
class SingletonBase
{
public:
    static T& instance()
    {
        static T obj;                      // 每个 T 独立一份
        return obj;
    }
};

class Config : public SingletonBase<Config> { /* ... */ };
Config::instance();                        // 只有一份 Config
```

> [!CAUTION] 单例的代价
> 单例是"优雅的全局变量"，会带来隐式依赖、测试困难、构造顺序问题。嵌入式里优先用**显式依赖注入**（引用/句柄传参）；确需全局唯一时才用单例，且避免在单例里放大对象（BSS/栈压力）。CRTP 原理见 [[模板与泛型编程]] §4。

---

## 🚫 4. `= delete` / `= default` / `explicit`

### 4.1 `= delete`：禁止某操作

```cpp
class A
{
public:
    A(int x) : v(x) {}
    A(const A&) = delete;                  // 禁止拷贝构造
    A& operator=(const A&) = delete;       // 禁止拷贝赋值
    int v;
};

A a(1);
A b = a;                                   // 编译错误
```

### 4.2 `= default`：明确"用编译器默认的"

```cpp
class B
{
public:
    B() = default;                         // 明确使用默认构造
    ~B() = default;                        // 明确使用默认析构
};
```

### 4.3 `explicit`：禁止单参构造的隐式转换

```cpp
class Celsius
{
public:
    explicit Celsius(double deg) : m_deg(deg) {}
private:
    double m_deg;
};

Celsius c = 30.0;                          // 错误：explicit 禁止隐式转换
Celsius c2(30.0);                          // 正确：显式构造
```

> [!TIP] 三者配合
> `= delete` 关掉不想要的操作、`= default` 保留编译器生成的操作、`explicit` 防止意外隐式转换。三者是写"值语义清晰、误用即编译错"的类型的基本功。

---

## 🔗 思考与延伸索引

- [[类型擦除与轻量 RTTI]] — `std::function` 的类型擦除机制与运行期开销
- [[模板与泛型编程]] — 模板回调（写法 C）与 CRTP 单例的模板基础
- [[ETL容器与萃取]] — `etl::function`（定容无堆回调）替代 `std::function`
- [[枚举constexpr多态与GCC扩展]] — 用 `enum` + 分派替代部分回调场景

---

## 📚 参考资料

1. **cppreference.com** — *Lambda expressions* / *std::function* / *Singleton* / *explicit*.
2. **Scott Meyers** — *Effective Modern C++*（Item 31–34：lambda、捕获、`std::function`）.
3. **Gamma et al.** — *Design Patterns*（Singleton 模式）.
4. **ISO/IEC 14882:2017** — *C++17 Standard*（Magic Static、`= delete`/`= default`）.
