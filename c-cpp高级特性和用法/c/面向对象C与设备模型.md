---
title: 面向对象 C 与设备模型教学手册（vtable / container_of / ERR_PTR / 宏代码生成）
tags:
  - C
  - OOP
  - Device-Model
  - Function-Pointer
  - container-of
  - Embedded
  - Teaching
standard: C11 + GNU 扩展
status: Completed
created: 2026-09-20
---

# 🏗️ 面向对象 C 与设备模型教学手册

> [!INFO] 笔记概览
> - **核心命题**：纯 C 用"结构体 + 函数指针表"就能实现封装、多态、抽象接口——`struct file_operations` 就是手写虚表，`container_of` 就是 C 版 `this`，`ERR_PTR` 就是替代异常的错误通道。
> - **内容范围**：vtable 运行时多态 · `container_of` 反推外层对象 · `ERR_PTR`/`IS_ERR` 指针夹带错误码 · 宏 `##` 代码生成 · 静态设备表 + 按名查找 · `constructor(N)` 初始化次序
> - **推荐路径**：§1 vtable → §2 container_of → §3 ERR_PTR → §4 宏生成 → §5 设备表 → §6 初始化次序
> - **代码约定**：最原生 C（标准 C + GNU 扩展），去掉工程自定义命名；**Allman 大括号、4 空格缩进、控制语句强制花括号**。

---

## 🎭 1. 用操作函数表（vtable）实现"运行时多态"

核心：结构体里存函数指针，让不同后端（UART/SPI/ADC…）挂进同一套接口，上层只认 `struct device*`：

```c
typedef struct device device_t;

struct file_operations
{
    int (*open)(device_t *dev);
    int (*close)(device_t *dev);
    int (*write)(device_t *dev, const void *buf, size_t len);
    int (*read)(device_t *dev, void *buf, size_t len);
};

struct device
{
    const struct file_operations *ops;   /* "多态表" */
    const char                   *name;  /* 设备名，供按名查找 */
    void                         *priv;  /* 驱动私有态 */
};
```

> [!NOTE] 类比 C++
> `struct file_operations` ≈ 虚表 `vptr`；`dev->ops->read(...)` ≈ 调虚函数，只是 C 里手动分发。没有编译器生成的 vtable 开销，分发路径完全可见。

某个具体驱动填自己的表（用**指定初始化器**最直观）：

```c
static int uart_open(device_t *d)
{
    /* ... */
    return 0;
}

static int uart_write(device_t *d, const void *b, size_t n)
{
    /* ... */
    return (int)n;
}

static const struct file_operations uart_ops =
{
    .open  = uart_open,
    .write = uart_write,
};

/* 初始化时：d->ops = &uart_ops; */
```

---

## 🔙 2. `container_of` 取回"驱动私有对象"（C 版的 this）

回调里只拿到 `device_t*`，真正的私有状态在"内嵌结构体"里，用 `container_of` 反推：

```c
#include <stddef.h>

#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))

typedef struct
{
    device_t dev;      /* 内嵌在开头，偏移常为 0 */
    int      sampled;
} adc_dev_t;

/* 回调里： */
static int adc_ioctl(device_t *d, void *arg)
{
    adc_dev_t *self = container_of(d, adc_dev_t, dev);  /* 拿到外层对象 */
    self->sampled = da_read();                          /* 访问私有状态 */
    return 0;
}
```

> [!TIP] 内嵌在开头的好处
> "设备内嵌在驱动状态结构体的开头"是常见布局，这样 `container_of` 的偏移经常是 0，甚至可以直接强转。带类型检查的内核版 `container_of` 见 [[嵌入式内核]] §1。

---

## ⚠️ 3. `ERR_PTR / IS_ERR`：指针里"夹带"错误码

某些函数要"返回指针"又要"可能失败"，把错误码编码进指针的高位地址区：

```c
struct device *device_find(const char *name)
{
    if (!found)
    {
        return ERR_PTR(-ENODEV);    /* 返回"错误指针" */
    }
    return the_device;              /* 成功返回真指针 */
}

void use(const char *name)
{
    device_t *d = device_find(name);
    if (IS_ERR(d))                  /* 判断是不是错误指针 */
    {
        int err = PTR_ERR(d);       /* 还原错误码 */
        log("err %d", err);
        return;
    }
    d->ops->read(d, buf, n);
}
```

> [!NOTE] 原理
> 约定"合法指针都落在某个基址之下"，把错误码放在该基址之上，靠指针数值大小同时表示"成功指针 或 错误码"。这是 Linux 内核经典手法，替代异常（嵌入式禁异常）。本项目统一用 `VFS_OK`/`VFS_ERR_*` 与 `ERR_PTR`/`PTR_ERR`/`IS_ERR`/`IS_ERR_OR_NULL`。

---

## 🏭 4. 宏生成样板：把"注册"展开成函数对

用 `##` 拼接函数名，一次声明展开出多个符号：

```c
#define REGISTER_DRIVER(name, probe_fn, remove_fn)                       \
    int driver_probe_##name(struct device *d)  { return probe_fn(d); }   \
    int driver_remove_##name(struct device *d) { return remove_fn(d); }

REGISTER_DRIVER(adc, adc_probe, adc_remove);   /* 展开成 driver_probe_adc / driver_remove_adc */
```

> [!TIP] 宏的代码生成用途
> 样板代码由编译器代为展开，避免手写重复的注册函数。`##` 拼接、`do-while(0)`、`#__VA_ARGS__` 等宏技巧见 [[C语言的宏魔法]]。

---

## 📇 5. 一张设备表 + 按名字查找（服务定位）

设备全部静态注册进一张表，提供按名字取回：

```c
static device_t s_devices[MAX_DEVICE];
static dev_id   s_used[MAX_DEVICE];        /* 记录占用 */

device_t *device_find(const char *name)
{
    for (dev_id i = 0; i < MAX_DEVICE; ++i)
    {
        if (s_used[i] && strcmp(s_devices[i].name, name) == 0)
        {
            return &s_devices[i];
        }
    }
    return ERR_PTR(-ENODEV);
}
```

> [!NOTE] 零堆标配
> 不 `malloc`、编译期定长，配合位图/used 标记管理占用——嵌入式零堆标配。还可给设备多个"别名"维度（名字 / 标签 / 兼容名 compatible）逐级查找，与 DTS 的 `compatible` 匹配同源。

---

## ⏱️ 6. `__attribute__((constructor(N)))` 控制初始化次序

```c
__attribute__((constructor(200))) void adc_pool_init(void);    /* 先跑 */
__attribute__((constructor(300))) void adc_driver_init(void);  /* 后跑 */
```

> [!TIP] 优先级编排
> `constructor(N)` 可指定优先级（数值小的先执行），精确编排多个自动初始化的先后，免去 `main` 里手排 `xxx_init()` 序列。本项目启动期初始化用 `pre_execution(priority)` 属性（外设 50、驱动池 160）。

---

## 📊 7. 小结：C 手法 ↔ C++ 概念

| C 手法 | 对应 C++ 概念 |
| :--- | :--- |
| `struct file_operations` 函数指针表 | 虚表 / 虚函数多态 |
| `container_of` | 从 `this` 反推对象 |
| `ERR_PTR` / `IS_ERR` | 返回值异常通道（替代 exception） |
| 宏 `##` 代码生成 | 样板 / 模板 |
| 静态设备表 + 按名查找 | 依赖注入 + 服务定位 |
| `constructor(N)` 属性 | 控制初始化次序 |

---

## 🔗 思考与延伸索引

- [[嵌入式内核]] — 带类型检查的 `container_of` 内核版、位图调度、优先级继承
- [[C语言的宏魔法]] — `##` 拼接、`do-while(0)`、`_Static_assert` 等宏技巧
- [[C语言高级用法]] — 指定初始化器、数组长度宏、断言宏等基础
- [[类型擦除与轻量 RTTI]] — C++ 侧用类型擦除实现同样的"统一接口 + 分发"

---

## 📚 参考资料

1. **Linux Kernel** — *Documentation/process/coding-style.rst*（`container_of`、`ERR_PTR` 手法）.
2. **《Linux 设备驱动程序》（LDD3）** — 设备模型、`file_operations`、cdev.
3. **GNU C Manual** — *Statement Exprs* / *Attribute Syntax*（`constructor`、`weak`）.
4. 项目内部笔记：[[嵌入式内核]]、[[C语言的宏魔法]]。
