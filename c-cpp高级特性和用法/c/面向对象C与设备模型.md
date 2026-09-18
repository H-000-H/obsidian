# 面向对象 C 与设备模型

用最原生 C 结构体 + 函数指针讲解"纯 C 怎么实现封装、多态、抽象接口"，去掉工程自定义命名。

## 1. 用操作函数表（vtable）实现"运行时多态"

核心：结构体里存函数指针，让不同后端（UART/SPI/ADC…）挂进同一套接口，上层只认 `struct device*`：

```c
typedef struct device device_t;

struct file_operations {
    int  (*open)(device_t* dev);
    int  (*close)(device_t* dev);
    int  (*write)(device_t* dev, const void* buf, size_t len);
    int  (*read)(device_t* dev, void* buf, size_t len);
};

struct device {
    const struct file_operations* ops;   // "多态表"
    void*       priv;                    // 驱动私有态
};
```

**类比 C++**：`struct file_operations` ≈ 虚表 `vptr`；`dev->ops->read(...)` ≈ 调虚函数，只是 C 里手动分发。

某个具体驱动填自己的表（用**指定初始化器**最直观）：

```c
static int uart_open(device_t* d) { /* ... */ }
static int uart_write(device_t* d, const void* b, size_t n) { /* ... */ }

static const struct file_operations uart_ops = {
    .open  = uart_open,
    .write = uart_write,
};
// 初始化时：d->ops = &uart_ops;
```

## 2. `container_of` 取回"驱动私有对象"（C 版的 this）

回调里只拿到 `device_t*`，真正的私有状态在"内嵌结构体"里，用 `container_of` 反推：

```c
#include <stddef.h>
#define container_of(ptr, type, member) \
    ((type*)((char*)(ptr) - offsetof(type, member)))

typedef struct { device_t dev; int sampled; } adc_dev_t;

// 回调里:
static int adc_ioctl(device_t* d, void* arg)
{
    adc_dev_t* self = container_of(d, adc_dev_t, dev);  // 拿到外层对象
    self->sampled = da_read();                           // 访问私有状态
    return 0;
}
```

"设备内嵌在驱动状态结构体的开头"是常见布局，这样 `container_of` 的偏移经常是 0，甚至可以直接强转。

## 3. `ERR_PTR / IS_ERR`：指针里"夹带"错误码

某些函数要"返回指针"又要"可能失败"，把错误码编码进指针的高位地址区：

```c
struct device* device_find(const char* name)
{
    if (!found)
        return ERR_PTR(-ENODEV);    // 返回"错误指针"
    return the_device;              // 成功返回真指针
}

void use(const char* name)
{
    device_t* d = device_find(name);
    if (IS_ERR(d)) {                // 判断是不是错误指针
        int err = PTR_ERR(d);       // 还原错误码
        log("err %d", err);
        return;
    }
    d->ops->read(d, buf, n);
}
```

**原理**：约定"合法指针都落在某个基址之下"，把错误码放在该基址之上，靠指针数值大小同时表示"成功指针 或 错误码"。Linux 内核经典手法。

## 4. 宏生成样板：把"注册"展开成函数对

用 `##` 拼接函数名，一次声明展开出多个符号：

```c
#define REGISTER_DRIVER(name, probe_fn, remove_fn)                       \
    int driver_probe_##name(struct device* d)  { return probe_fn(d); }   \
    int driver_remove_##name(struct device* d) { return remove_fn(d); }

REGISTER_DRIVER(adc, adc_probe, adc_remove);   // 展开成 driver_probe_adc / driver_remove_adc
```

这是宏的"代码生成"用途：样板代码由编译器代为展开。

## 5. 一张设备表 + 按名字查找（服务定位）

设备全部静态注册进一张表，提供按名字取回：

```c
static device_t s_devices[MAX_DEVICE];
static dev_id    s_used[MAX_DEVICE];       // 记录占用

device_t* device_find(const char* name)
{
    for (dev_id i = 0; i < MAX_DEVICE; ++i)
        if (s_used[i] && strcmp(s_devices[i].name, name) == 0)
            return &s_devices[i];
    return ERR_PTR(-ENODEV);
}
```

不 malloc、编译期定长，配合位图/used 标记管理占用——嵌入式零堆标配。还给设备多个"别名"维度（名字 / 标签 / 兼容名）可逐级查找。

## 6. `__attribute__((constructor(N)))` 控制偏初始化次序

```c
__attribute__((constructor(200))) void adc_pool_init(void);   // 先跑
__attribute__((constructor(300))) void adc_driver_init(void); // 后跑
```

`constructor(N)` 可指定优先级，精确编排多个自动初始化的先后。

## 小结

| 手法 | 对应 C++ 概念 |
| --- | --- |
| `struct file_operations` 函数指针表 | 虚表 / 虚函数多态 |
| `container_of` | 从 `this` 反推对象 |
| `ERR_PTR / IS_ERR` | 返回值异常通道（替代 exception） |
| 宏 `##` 代码生成 | 样板 / 模板 |
| 静态设备表 + 按名查找 | 依赖注入 + 服务定位 |
| `constructor(N)` 属性 | 控制初始化次序 |

## 关联
- [[嵌入式内核]]（container_of 内核版）
- [[C 语言的宏魔法]]（`##`、constructor）