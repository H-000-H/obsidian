
- new(destination) F(static_cast<const F*>(src)) 强行求具体的类型加直接在destination进行覆盖不调用底层malloc
```
    template <typename F>
    static void copy_fn(void* dst, const void* src)
    {
        new (dst) F(*static_cast<const F*>(src));
    }
```
- !etl::is_same_v<etl::decay_t<F>, CmdFn>判断退化后的 `F` 是不是当前类 `CmdFn` 自身，如果是外部传入的 lambda 或函数指针，这里是 `true`；如果传入的是另一个 `CmdFn` 对象，这里是 `false` 。
- etl::decay_t<F> 把传入的类型 `F` 退化为最基础的原始类型（去除引用 `&`、`&&` 以及顶层 `const`）
- etl::enable_if_t<条件, 类型 = void>当条件为 `true` 时，该表达式被替换为 `void`，模板参数变成 `typename = void`，模板实例化成功，构造函数参与重载候选。当条件为 `false` 时，该表达式**无定义**。根据 SFINAE 原则，编译器不会报错，而是默默地将这个构造函数从候选列表中剔除。