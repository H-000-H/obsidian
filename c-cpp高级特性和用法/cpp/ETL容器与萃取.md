---
title: 容器与类型萃取教学手册（零堆嵌入式版：array / span / optional / function / type_traits）
tags:
  - CPP
  - ETL
  - Container
  - Type-Traits
  - Embedded
  - Zero-Heap
  - Teaching
standard: C++17
status: Completed
---

# 🧰 容器与类型萃取教学手册（零堆嵌入式版）

> [!INFO] 笔记概览
> - **核心命题**：嵌入式禁用堆，但容器/萃取的**概念与标准库完全一致**——只是"大小编译期定死、不 new 内存"。本笔记用标准 `std` 语法讲清每个容器能不能、怎么在嵌入式里用，并给出对应的 ETL 零堆实现。
> - **内容范围**：`array`（定长）· `span`（只读视图）· `optional`（可能无值）· `function`（类型安全回调）· `<type_traits>`（编译期类型侦探）· `forward`/`move` · std↔ETL 映射
> - **推荐路径**：§0 能否用 std → §1 array → §2 span → §3 optional → §4 function → §5 type_traits → §6 move/forward → §7 ETL 映射 → §8 小结
> - **代码约定**：标准 C++17 原生写法；大括号遵循 Allman（控制语句强制花括号）。

---

## ❓ 0. 嵌入式能用 std 容器吗？

标准 `std::vector` / `std::string` 会在**堆上动态扩容**，嵌入式往往禁用堆分配，不能直接用。对策有两条：

- 用**定长容器**：`std::array`（栈/静态分配、固定大小，能直接用）。
- 用标准容器但**预留足量并禁止扩容**，或用等价的"零堆实现"（接口等同，只是固定大小、不 `new`）——这正是 **ETL** 的定位。

> [!NOTE] 本质一致
> 核心概念和标准库完全一致，只是"大小编译期定死、不分配内存"。学会 `std` 语义，切到 `etl::` 几乎零成本。

---

## 📏 1. `std::array`：定长数组 + 安全接口

```cpp
#include <array>
#include <cstdint>

std::array<uint8_t, 16384> buf{};   // 固定 16KB，栈/静态分配，{} 值初始化
buf.size();                         // 16384
buf[i];                             // 无检查访问（嵌入式首选，零开销）
// 还能配合范围 for 遍历
```

> [!CAUTION] `.at()` 会抛异常
> `buf.at(i)` 越界会**抛异常**（`std::out_of_range`），而嵌入式通常关闭异常（`-fno-exceptions`）。因此嵌入式优先用 `operator[]` 或 ETL 的错误处理回调，别依赖 `.at()`。底层就是 `uint8_t buf[16384]`，零开销。

---

## 👁️ 2. `std::span`：只读视图，不拥有数据

```cpp
#include <span>
#include <cstdint>

void process(std::span<const uint8_t> data)   // 指针 + 长度，不拷贝
{
    for (auto b : data)
    {
        /* ... */
    }
}

uint8_t rx[256];
process({rx, actual_len});                    // 只传"借看这一段"
```

> [!TIP] 告别 (ptr, len) 双参数
> `span` 持有"指针 + 长度"，语义是"我借用它、不拥有它"。避免到处传 `(ptr, len)` 两个参数或悄悄拷贝。缓冲传参首选 `span` / `string_view`（带长度零拷贝）。

---

## 🎁 3. `std::optional`：表示"可能没有值"

```cpp
#include <optional>
#include <iostream>

std::optional<int> maybe = 42;
if (maybe.has_value())
{
    std::cout << maybe.value();
}
maybe.reset();                        // 清空，变成"无值"

std::optional<float> empty;           // 默认就是无值
```

> [!NOTE] 类型安全的"有无"
> 比"NULL 指针 / -1 魔法值"类型安全，杜绝哨兵歧义；`emplace()` 可就地构造，不先做默认构造。返回值优先 `optional<T>` 显式表达有无值。值语义、无堆，嵌入式可用。

---

## 🔔 4. 回调用 `std::function`（类型安全，替代裸函数指针）

```cpp
#include <functional>

using Callback = std::function<bool(int)>;    // 签名固定

Callback cb = [](int v) { return v > 0; };    // 能装 lambda
bool ok = cb(3);
```

> [!WARNING] std::function 可能用堆
> 签名写死在类型里、类型不对编译就拒绝，还能绑定带状态的 lambda——但 `std::function` 对大捕获可能**堆分配**。嵌入式禁堆时用 **`etl::function`（定容、SBO、无堆）** 或直接用函数指针。类型擦除原理见 [[类型擦除与轻量 RTTI]]。

---

## 🕵️ 5. 类型萃取 `<type_traits>`：编译期类型侦探

配合 `if constexpr` / `static_assert`，编译期推导类型属性：

```cpp
#include <type_traits>

std::is_same_v<int, int>;                    // true：两类型是否相同
std::decay_t<const int&>;                    // int：去掉引用和顶层 const
std::is_trivially_copyable_v<MyPod>;         // bool：能否直接 memcpy
```

### 5.1 `std::decay`：把类型"退化"回裸形态

```cpp
using T = std::decay_t<const int&&>;         // int
```

去掉 `&` / `&&` / 顶层 `const`，用于统一比较类型、写泛型逻辑。

### 5.2 `is_trivially_copyable`：能安全 memcpy 吗

```cpp
static_assert(std::is_trivially_copyable_v<Header>, "Header 必须可平凡复制才能 memcpy");
```

> [!TIP] 编译期把"约定"写进代码
> 编译期就知道"这类能否直接按字节搬"，从而放心在临界区/通信缓冲区里 `memcpy`。传错类型时是**编译错**而非运行崩。`<type_traits>` 是纯头文件、无运行时代码，嵌入式直接用。

---

## 🏓 6. `std::forward` 与 `std::move`

```cpp
#include <utility>

template <typename T>
void relay(T&& v)
{
    use(std::forward<T>(v));         // 保持 v 原本的左/右值性质
}

auto p = std::move(someObject);      // 显式转成右值，触发移动而非拷贝
```

> [!NOTE] 别滥用 std::move
> 若对象本身就按值持有或走 C++17 复制省略，很多场景甚至不需要显式 `std::move`。移动语义水平不足时易出内存问题，优先 `const T&`。完美转发见 [[模板与泛型编程]] §7。

---

## 🔁 7. std ↔ ETL 映射（嵌入式落地）

本项目上层业务用 **ETL**（定容、无堆）替代堆 STL。概念一一对应：

| 标准库（教学） | ETL（嵌入式落地） | 是否零堆 | 说明 |
| :--- | :--- | :---: | :--- |
| `std::array<T,N>` | `etl::array<T,N>` | ✅ | 定长数组，可 constexpr |
| `std::span<T>` | `etl::span<T>` | ✅ | 非拥有零拷贝视图 |
| `std::optional<T>` | `etl::optional<T>` | ✅ | 表达可有可无 |
| `std::function<Sig>` | `etl::function<Sig>` | ✅ | 定容回调（SBO，无堆） |
| `std::vector<T>` | `etl::vector<T,N>` | ✅ | 定容动态数组 |
| `std::map<K,V>` | `etl::map<K,V,N>` | ✅ | 有序键值查找 |
| `std::string` | `etl::string<N>` | ✅ | 定长字符串 |
| `<type_traits>` | 同（ETL 复用 std 或自带） | ✅ | 编译期萃取，无运行时 |

> [!TIP] 构建约束
> ETL 按 `ETL_NO_STL` 约束构建，不引入 `std::vector`/`std::string` 等堆 STL。容器清单与用法见项目规则与 [[etl]]。

---

## 📊 8. 小结

| 用什么 | 一句话 | 能不能直接嵌入式用 |
| :--- | :--- | :--- |
| `array` | 定长数组 + 安全接口 | ✅ 能（零堆） |
| `span` | 借用一段数据的只读视图 | ✅ 能（零堆） |
| `optional` | 可能没值 | ✅ 能（值语义，无堆） |
| `function` | 类型安全回调 | ⚠️ 可能堆，用 `etl::function` 定容版 |
| `<type_traits>` | 编译期类型侦探 | ✅ 能（头文件，无运行时） |
| `forward` / `move` | 完美转发 / 移动 | ✅ 能（勿滥用 move） |

---

## 🔗 思考与延伸索引

- [[类型擦除与轻量 RTTI]] — `function` 的类型擦除机制与零堆 SBO 实现
- [[模板与泛型编程]] — NTTP 定容容器、完美转发与 `type_traits` 的模板基础
- [[etl]] — ETL 库总览与本项目容器清单
- [[Lambda 回调与单例模式]] — `etl::function` + lambda 回调的实战

---

## 📚 参考资料

1. **cppreference.com** — *std::array* / *std::span* / *std::optional* / *std::function* / *<type_traits>*.
2. **ETL 官方文档** — *Embedded Template Library*（etlcpp.com，定容无堆容器）.
3. **Scott Meyers** — *Effective Modern C++*（Item 3/7/8/10：`auto`、移动、`optional` 思想）.
4. **ISO/IEC 14882:2017/2020** — *C++17/20 Standard*（`span` 为 C++20，ETL 提供 C++11 等价）.
