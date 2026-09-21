---
title: ETL 类型萃取与 SFINAE 技巧手册（placement new / decay_t / is_same_v / enable_if_t）
tags:
  - CPP
  - ETL
  - Type-Traits
  - SFINAE
  - Placement-New
  - Type-Erasure
  - Embedded
  - Teaching
standard: C++17
status: Completed
created: 2026-09-20
---

# 🧬 ETL 类型萃取与 SFINAE 技巧手册

> [!INFO] 笔记概览
> - **核心命题**：ETL 的零堆类型擦除（`etl::function` / 命令包装器）靠四块基石拼成——**placement new 就地构造**（不 malloc）、**`decay_t` 类型退化**、**`is_same_v` 区分自身与外部可调用对象**、**`enable_if_t` + SFINAE 编译期开关重载**。
> - **内容范围**：placement new 拷贝构造 · `etl::decay_t` · `etl::is_same_v` · `etl::enable_if_t` 与 SFINAE · 组合成一个零堆命令包装器 `CmdFn`
> - **推荐路径**：§1 placement new → §2 decay → §3 is_same → §4 enable_if/SFINAE → §5 组合
> - **代码约定**：ETL（`etl::` 命名空间）+ 标准 C++17；大括号遵循 Allman。

---

## 🧱 1. placement new：零堆就地构造（类型擦除的基石）

类型擦除要把"任意大小的可调用对象"存进一块**固定大小的缓冲**，不能用 `malloc`。placement new 直接在给定地址上构造对象：

```cpp
template <typename F>
static void copy_fn(void* dst, const void* src)
{
    // 强行求出具体类型 F，直接在 dst 上就地拷贝构造，不调用底层 malloc
    new (dst) F(*static_cast<const F*>(src));
}
```

> [!NOTE] 为什么能"零堆"
> `new (dst) F(...)` 是 placement new：不分配内存，只在 `dst` 指向的**已有缓冲**上调用 `F` 的拷贝构造。`src` 是 `const void*`，先 `static_cast<const F*>(src)` 还原成具体类型再拷贝。这正是 `etl::function` 的 SBO（小缓冲优化）核心，见 [[类型擦除与轻量 RTTI]] §1.3。

---

## 🔧 2. `etl::decay_t`：把类型"退化"回裸形态

`etl::decay_t<F>` 把传入的类型 `F` 退化为最基础的原始类型——去除引用 `&`、`&&` 以及顶层 `const`：

```cpp
etl::decay_t<const int&>   // → int
etl::decay_t<int&&>        // → int
etl::decay_t<const Foo>    // → Foo
```

> [!TIP] 为什么要退化
> 万能引用 `F&&` 会推导出 `int&`、`const int&`、`int&&` 等各种形态。要"比较类型本身"或"按值存储"，必须先 `decay` 回裸类型，否则引用/const 会干扰判断。等价于 `std::decay_t`，见 [[ETL容器与萃取]] §5.1。

---

## 🆔 3. `etl::is_same_v`：区分自身与外部可调用对象

```cpp
etl::is_same_v<etl::decay_t<F>, CmdFn>
```

判断**退化后的 `F`** 是不是当前类 `CmdFn` 自身：

| 传入的 F | `is_same_v<decay_t<F>, CmdFn>` | 含义 |
| :--- | :---: | :--- |
| 外部 lambda / 函数指针 | `true`（是"别的可调用对象"） | 应包装它 |
| 另一个 `CmdFn` 对象 | `false`（就是自己） | 应走拷贝/移动构造 |

> [!CAUTION] 别把"拷贝自己"误当"包装回调"
> 若不加这个判断，`CmdFn b(a);`（拷贝另一个 `CmdFn`）会被模板构造函数当成"包装一个可调用对象"，导致语义错乱。用 `is_same_v` 把"自身类型"排除在包装构造之外。

---

## 🚦 4. `etl::enable_if_t` 与 SFINAE：编译期开关重载

`etl::enable_if_t<条件, 类型 = void>`：

- 条件为 **`true`** → 表达式被替换为 `void`，模板参数变成 `typename = void`，实例化成功，构造函数**参与重载候选**。
- 条件为 **`false`** → 该表达式**无定义**。根据 **SFINAE**（Substitution Failure Is Not An Error）原则，编译器**不报错**，而是默默把这个构造函数从候选列表中剔除。

```cpp
// 仅当 F 退化后不是 CmdFn 自身时，才启用这个"包装可调用对象"的构造函数
template <typename F,
          typename = etl::enable_if_t<!etl::is_same_v<etl::decay_t<F>, CmdFn>>>
CmdFn(F&& f)
{
    // ... 用 placement new 把 f 存进内部缓冲
}
```

> [!NOTE] SFINAE 的本质
> 模板实参替换失败**不是编译错误**，只是让该候选"静默出局"。于是可以用编译期条件精确控制"哪个构造函数在什么类型下可见"，实现类型安全的重载分派。

---

## 🏗️ 5. 组合：一个零堆命令包装器 `CmdFn`

把四块基石拼起来，就是一个定容、无堆、类型安全的命令包装器骨架：

```cpp
class CmdFn
{
public:
    // ① 包装"外部可调用对象"：decay 后不是 CmdFn 自身才启用（enable_if + SFINAE）
    template <typename F,
              typename = etl::enable_if_t<!etl::is_same_v<etl::decay_t<F>, CmdFn>>>
    CmdFn(F&& f)
    {
        using Fn = etl::decay_t<F>;
        static_assert(sizeof(Fn) <= sizeof(m_storage), "可调用对象超出定容缓冲");
        // ② placement new 就地构造，不 malloc
        new (&m_storage) Fn(etl::forward<F>(f));
        m_invoke = [](const void* p) { (*static_cast<const Fn*>(p))(); };
    }

    // ③ 拷贝另一个 CmdFn：走各自的拷贝构造（is_same 为 false 时不参与上面的模板）
    CmdFn(const CmdFn& other) { /* 通过 other.m_invoke/m_storage 深拷贝 */ }

    void operator()() const { m_invoke(&m_storage); }

private:
    alignas(max_align_t) unsigned char m_storage[16];  // 定容 SBO 缓冲
    void (*m_invoke)(const void*);                     // 类型擦除后的统一调用
};
```

> [!TIP] 这就是 etl::function 的原理
> 定容缓冲 + placement new + 函数指针 invoke + SFINAE 构造分派 = 零堆类型擦除。`static_assert(sizeof(Fn) <= sizeof(m_storage))` 在编译期拦住"捕获过大撑爆缓冲"的情况。完整机制见 [[类型擦除与轻量 RTTI]]、[[ETL容器与萃取]] §4。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 用 ETL 萃取 / SFINAE 前逐条核对
> - [ ] **placement new**：在已有缓冲上构造，配对调用析构，不 `malloc`/`delete`
> - [ ] **缓冲对齐**：SBO 缓冲用 `alignas(max_align_t)`，避免未对齐访问
> - [ ] **容量兜底**：`static_assert(sizeof(Fn) <= sizeof(storage))` 编译期拦超大捕获
> - [ ] **先 decay 再比较**：类型判断前用 `decay_t` 去掉引用/顶层 const
> - [ ] **排除自身**：包装构造用 `!is_same_v<decay_t<F>, Self>` 防误吞拷贝构造
> - [ ] **SFINAE 开关**：用 `enable_if_t` 控制重载可见性，失败静默剔除而非报错
> - [ ] **forward 保值类别**：存储时 `etl::forward<F>(f)`，不丢左/右值性质

---

## 🔗 思考与延伸索引

- [[类型擦除与轻量 RTTI]] — placement new + invoke 指针构成的类型擦除全貌
- [[ETL容器与萃取]] — `etl::function` 定容回调与 `type_traits` 萃取清单
- [[模板与泛型编程]] — 万能引用、完美转发、`if constexpr` 与 SFINAE 的关系
- [[Lambda 回调与单例模式]] — 被包装的 lambda / 回调的三种写法

---

## 📚 参考资料

1. **ETL 官方文档** — *etl::function* / *Type Traits*（etlcpp.com）.
2. **cppreference.com** — *Placement new* / *std::decay* / *std::enable_if* / *SFINAE*.
3. **Sean Parent** — *Inheritance Is The Base Class Of Evil*（类型擦除与 SBO）.
4. **ISO/IEC 14882:2017** — *C++17 Standard*（`<type_traits>`、SFINAE、`alignas`）.
