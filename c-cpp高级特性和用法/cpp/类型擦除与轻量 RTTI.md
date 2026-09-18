# 类型擦除与轻量 RTTI

用标准 C++（std 家族）的最小示例讲解，能看懂就行。

## 一、类型擦除：把"不同类型的东西"统一装进一个容器

### 1.1 问题
回调、事件处理器常是不同类型：函数指针、无捕获 lambda、带状态的 lambda……占的内存不同、类型不同，却想统一存进一个数组。

### 1.2 正解：统一签名的函数对象（std::function）

`std::function` 把"随便什么可调用对象"擦成同一签名，怎么来的不重要，都能一致地 `f()` 调用：

```cpp
#include <functional>
std::function<void(int)> f;

f = [](int x) { std::cout << x; };          // lambda
f = someFreeFunction;                        // 函数指针
f(42);                                       // 统一的调用，不论当初塞的是什么
```

这就是**类型擦除**：调用方只需"能调它"，不必知道它内部是什么类型。

### 1.3 底层原理（为什么 std::function 能装下不同大小）

`std::function` 内部靠"一个模板化的仿函数对象 + 一对 `copy`/`invoke` 函数指针"实现，本质是：

```cpp
struct FunctionModel
{
    void (*invoke)(const void*);
    void* self;
};
```

模板代码把每个（不同大小的）callable 构造，把"怎么调它"提取成一个函数指针，放进统一结构——于是既能统一存、又能正确调。

> 注：std::function 可能用堆；嵌入式禁止堆时用同原理的固定容量版本（SBO）。概念完全一致。

## 二、轻量 RTTI：不靠编译器 RTTI，自己给类型做"指纹"

### 2.1 背景
标准库有两套判断类型的方式：
- 完整 RTTI（`typeid` / `dynamic_cast`）：灵活，但体积大，很多嵌入式工程用 `-fno-rtti` 关掉。
- 模板 `type_traits`：编译期确定、零开销，但只在编译期。

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

需要时把"期望类型"和"实际类型"的两个指针地址一比即可——零 vtable、零 RTTI、O(1)，这就是轻量 RTTI。

## 三、编译期契约：`static_assert`

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

`std::is_trivially_copyable_v<T>` 编译期就知道"T 能否直接被 memcpy"，于是可放心搬移。作用：把"约定"写进代码，传错类型时是**编译错**而非运行崩。

## 四、为什么把这三样放一起

一条命令分发器 = 三个技巧组合：
1. **类型擦除**（函数指针 / function）→ 把所有处理器统一存进一个数组；
2. **轻量 RTTI**（模板静态变量的地址指纹）→ 找出每个处理器对应类型，分发时比对防用错；
3. **`static_assert`** → 编译期约束参数必须平凡可复制。

合起来达到：用一个固定大小的静态表，类型安全地分发任意命令，且零堆、零 RTTI。

## 关联
- [[模板与泛型编程]]