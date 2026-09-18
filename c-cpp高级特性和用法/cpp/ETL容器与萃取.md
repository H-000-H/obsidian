# 容器与类型萃取（零堆嵌入式版）

用标准 `std` 语法讲清这些容器能不能 / 怎么在嵌入式里用，去掉别名包装。

## 0. 嵌入式能用 std 容器吗？

标准 `std::vector/std::string` 会在堆上动态扩容量，嵌入式往往禁用堆分配，不能直接用。对策有两条：
- 用**定长容器**：`std::array`（栈上、固定大小，能直接用）。
- 用标准容器但**预留足量**并禁止扩容（或用等价的"零堆实现"——接口等同，只是固定大小、不 new）。

本质：核心概念和标准库完全一致，只是"大小编译期定死、不分配内存"。

## 1. `std::array`：定长数组 + 安全的接口

```cpp
#include <array>
std::array<uint8_t, 16384> buf{};   // 固定 16KB, 栈/静态分配
buf.size();                         // 16384
buf.at(i);                          // 越界抛异常/断言
// 还能配合范围 for 遍历
```

要点：底层就是 `uint8_t buf[16384]`，零开销；`array` 只是包了一层 `size()/at()/begin()` 的好接口。嵌入式直接用合适。

## 2. `std::span`：只读视图，不拥有数据

```cpp
#include <span>
void process(std::span<const uint8_t> data)   // 指针+长度, 不拷贝
{
    for (auto b : data) { /* ... */ }
}

uint8_t rx[256];
process({rx, actual_len});    // 只传"借看这一段"
```

要点：`span` 持有"指针+长度"，表示"我借用它、不拥有它"。避免到处传 `(ptr, len)` 两个参数或悄悄拷贝。

## 3. `std::optional`：表示"可能没有值"

```cpp
#include <optional>
std::optional<int> maybe = 42;
if (maybe.has_value())
    std::cout << maybe.value();
maybe.reset();                    // 清空, 变成"无值"
std::optional<float> empty;       // 默认就是无值
```

要点：比"NULL 指针 / -1 魔法值"类型安全；`emplace()` 可直接就地构造，不先做默认构造。

## 4. 回调用 `std::function`（类型安全，替代裸函数指针）

```cpp
#include <functional>
using Callback = std::function<bool(int)>;    // 签名固定

Callback cb = [](int v) { return v > 0; };    // 能装 lambda
bool ok = cb(3);
```

要点：签名写死在类型里，类型不对编译就拒绝；还能绑定带状态的 lambda。

## 5. 类型萃取 `<type_traits>`：编译期类型侦探

配合 `if constexpr` / `static_assert`，编译期推导类型属性：

```cpp
#include <type_traits>

std::is_same_v<int, int>;                    // true: 两类型是否相同
std::decay_t<const int&>;                    // int: 去掉引用和顶层 const
std::is_trivially_copyable_v<MyPod>;         // bool: 能否直接 memcpy
```

### 5.1 `std::decay`：把类型"退化"回裸形态
```cpp
using T = std::decay_t<const int&&>;   // int
```
去掉 `&`/`&&`/顶层 `const`，用于统一比较类型、写泛型逻辑。

### 5.2 `is_trivially_copyable`：能安全 memcpy 吗
```cpp
static_assert(std::is_trivially_copyable_v<Header>, "才行 memcpy");
```
编译期知道"这类能否直接按字节搬"，从而放心在临界区/通信缓冲区里搬。

## 6. `std::forward` 与 `std::move`

```cpp
template <typename T>
void relay(T&& v)
{
    use(std::forward<T>(v));    // 保持 v 原本的左/右值性质
}
auto p = std::move(someObject); // 显式转成右值, 触发移动而非拷贝
```

> 小提示：若对象本身就按值持有或走 C++17 复制省略，很多场景甚至不需要显式 `std::move`。

## 7. 小结

| 用什么 | 一句话 | 能不能直接嵌入式用 |
| --- | --- | --- |
| `std::array` | 定长数组 + 安全接口 | 能（零堆） |
| `std::span` | 借用一段数据的只读视图 | 能（零堆） |
| `std::optional` | 可能没值 | 能（值语义, 无堆） |
| `std::function` | 类型安全、零堆版替代裸指针 | 注意可能要堆, 用定容实现 |
| `<type_traits>` | 编译期类型侦探 | 能（头文件, 无运行时） |
| `std::forward` / `std::move` | 完美转发 / 移动 | 能 |

## 关联
- [[类型擦除与轻量 RTTI]]（std::function 的概念）