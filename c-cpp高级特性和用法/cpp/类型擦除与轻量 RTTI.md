---
title: C++ 类型擦除与轻量 RTTI 教学手册（std::function 原理 · 地址指纹 · static_assert 契约）
tags:
  - CPP
  - Type-Erasure
  - RTTI
  - std-function
  - Type-Traits
  - Embedded
  - Teaching
standard: C++17
status: Completed
created: 2026-09-20
---

# 🎭 C++ 类型擦除与轻量 RTTI 教学手册

> [!INFO] 笔记概览
> - **核心命题**：类型擦除让"不同类型的可调用对象"统一存进一个容器；轻量 RTTI 用"模板静态变量的唯一地址"做类型指纹，**零 vtable、零编译器 RTTI、O(1)** 完成运行期类型比对；`static_assert` 把契约写进编译期。
> - **内容范围**：类型擦除动机 · `std::function` 用法与底层原理（SBO）· 地址指纹轻量 RTTI · 给 handler 挂类型信息 · `static_assert` 编译期契约 · 三者组合成命令分发器
> - **推荐路径**：§1 类型擦除 → §2 轻量 RTTI → §3 static_assert → §4 组合应用
> - **代码约定**：标准 C++（std 家族）最小示例；大括号遵循 Allman。

---

## 🧩 1. 类型擦除：把"不同类型的东西"统一装进一个容器

### 1.1 问题

回调、事件处理器常是不同类型：函数指针、无捕获 lambda、带状态的 lambda……占的内存不同、类型不同，却想统一存进一个数组。

### 1.2 正解：统一签名的函数对象（`std::function`）

`std::function` 把"随便什么可调用对象"擦成同一签名，怎么来的不重要，都能一致地 `f()` 调用：

```cpp
#include <functional>
#include <iostream>

std::function<void(int)> f;

f = [](int x) { std::cout << x; };   // lambda
f = someFreeFunction;                // 函数指针
f(42);                               // 统一的调用，不论当初塞的是什么
```

> [!NOTE] 这就是类型擦除
> 调用方只需"能调它"，不必知道它内部是什么类型。类型信息被"擦掉"，只留下统一的调用接口。

### 1.3 底层原理（为什么 std::function 能装下不同大小）

`std::function` 内部靠"一个模板化的仿函数对象 + 一对 `copy`/`invoke` 函数指针"实现，本质是：

```cpp
struct FunctionModel
{
    void (*invoke)(const void*);   // 怎么调它（类型擦除后的统一入口）
    void* self;                    // 指向被存储的 callable
};
```

模板代码把每个（不同大小的）callable 构造，把"怎么调它"提取成一个函数指针，放进统一结构——于是既能统一存、又能正确调。

> [!WARNING] 嵌入式禁堆时用 SBO 定容版
> `std::function` 对大捕获可能用**堆**。嵌入式禁止堆时，用同原理的**固定容量版本（SBO，Small Buffer Optimization）**——即 `etl::function`，小对象内联存储、不 new，概念完全一致。见 [[ETL容器与萃取]] §4、§7。

---

## 🔍 2. 轻量 RTTI：不靠编译器 RTTI，自己给类型做"指纹"

### 2.1 背景

标准库有两套判断类型的方式：

| 方式 | 优点 | 缺点 |
| :--- | :--- | :--- |
| 完整 RTTI（`typeid`/`dynamic_cast`） | 灵活、运行期任意比对 | 体积大，嵌入式常 `-fno-rtti` 关掉 |
| 模板 `type_traits` | 编译期确定、零开销 | 只在编译期，运行期拿不到 |

轻量 RTTI 取两者之长：**运行期可比对，但零体积开销**。

### 2.2 技巧：利用"模板静态变量的唯一地址"

给每种类型做运行时指纹，办法很朴素——模板里放一个 `static` 变量，编译器让每种类型各占一份独立地址：

```cpp
template <typename T>
const void* type_marker()
{
    static char marker;       // 每种 T 各有一份独立 static
    return &marker;           // 地址即 T 的"唯一身份证"
}

bool same = (type_marker<int>() == type_marker<int>());     // true
bool diff = (type_marker<int>() == type_marker<double>());  // false
```

### 2.3 实际用途：给"任意 handler"挂类型信息，运行时比对

```cpp
template <typename T>
struct Handler
{
    const void* type = type_marker<T>();   // 记住被注册的类型
    void (*fn)(const void*);               // 类型擦除后的统一调用
};
```

> [!TIP] 零 vtable、零 RTTI、O(1)
> 需要时把"期望类型"和"实际类型"的两个指针地址一比即可——这就是轻量 RTTI。不依赖 `typeid`/`dynamic_cast`（省 Flash/RAM），符合本项目"禁 RTTI，用静态类型与 variant/枚举"的约定。

---

## 📜 3. 编译期契约：`static_assert`

想编译期就拦下不符合要求的类型，用 `static_assert` + `<type_traits>`：

```cpp
#include <type_traits>
#include <cstring>

template <typename T>
T read_copy(const void* raw)
{
    static_assert(std::is_trivially_copyable_v<T>,
                  "T 必须是可平凡复制的");   // 编译期约束
    T t;
    std::memcpy(&t, raw, sizeof(T));         // 才能安全 memcpy
    return t;
}
```

> [!NOTE] 把"约定"写进代码
> `std::is_trivially_copyable_v<T>` 编译期就知道"T 能否直接被 memcpy"，于是可放心搬移。传错类型时是**编译错**而非运行崩。萃取与 `optional`/`span` 见 [[ETL容器与萃取]] §5。

---

## 🏗️ 4. 为什么把这三样放一起

一条命令分发器 = 三个技巧组合：

```text
① 类型擦除（函数指针 / function）
      └─► 把所有处理器统一存进一个数组
② 轻量 RTTI（模板静态变量的地址指纹）
      └─► 找出每个处理器对应类型，分发时比对防用错
③ static_assert
      └─► 编译期约束参数必须平凡可复制
```

合起来达到：**用一个固定大小的静态表，类型安全地分发任意命令，且零堆、零 RTTI**。这正是嵌入式里替代"虚函数多态 + 动态注册"的高性能方案。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 用类型擦除 / 轻量 RTTI 前逐条核对
> - [ ] **禁堆**：`std::function` 大捕获会堆分配，嵌入式改 `etl::function`（SBO 定容）
> - [ ] **简单回调**：无状态回调优先裸函数指针，比 `function` 更省
> - [ ] **轻量 RTTI**：用模板静态变量地址做指纹，不用 `typeid`/`dynamic_cast`
> - [ ] **地址唯一性**：确认每种类型的 `static` 标记在各翻译单元内唯一（模板保证）
> - [ ] **编译期契约**：能 `static_assert` 的约束绝不留到运行期
> - [ ] **memcpy 前提**：`is_trivially_copyable_v<T>` 通过才按字节搬
> - [ ] **分发表定容**：命令分发表用固定大小静态数组，不在运行时扩容

---

## 🔗 思考与延伸索引

- [[模板与泛型编程]] — 类型擦除依赖的模板实例化、CRTP 静态多态对照
- [[ETL容器与萃取]] — `etl::function`（定容 SBO）与 `<type_traits>` 萃取
- [[Lambda 回调与单例模式]] — 回调三种写法（函数指针 / `std::function` / 模板）的取舍
- [[面向对象C与设备模型]] — C 侧用函数指针表实现同样的"类型擦除 + 分发"

---

## 📚 参考资料

1. **cppreference.com** — *std::function* / *Type erasure* / *static_assert* / *<type_traits>*.
2. **Sean Parent** — *Inheritance Is The Base Class Of Evil*（类型擦除经典演讲）.
3. **ETL 官方文档** — *etl::function*（SBO 定容回调，etlcpp.com）.
4. **ISO/IEC 14882:2017** — *C++17 Standard*（`is_trivially_copyable`、`static_assert`）.
