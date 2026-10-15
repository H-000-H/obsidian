---
title: 驱动实战：从裸模块到 platform_driver
aliases:
  - 手写驱动实战
  - out-of-tree 模块
  - kbuild 多目录
  - 驱动踩坑集
tags:
  - linux/kernel
  - driver-model
  - practice
  - kbuild
status: Completed
created: 2026-10-15
kernel: "运行内核 7.0.0-38-generic（Ubuntu，发行版 headers）"
arch: x86_64
---

# 🔨 驱动实战：从裸模块到 platform_driver

> [!INFO] 笔记定位
> - **性质**：**动手实战**笔记（对照前 10 篇源码精读）。目标是把 `kobject`/`cdev`/`device`/`bus`/`driver` 一路学到的概念，**亲手跑一遍**。
> - **路线**：裸模块 → 字符设备 + `/dev` → 多实例（1 cdev 管 4 次号）→ `platform_driver` + 假设备。
> - **最值钱的部分是 §5 踩坑集** —— 全是真机上撞出来的。
> - 关联：[[struct platform_driver 与 probe 时序]] · [[file_operations 与字符设备结构精读]] · [[sysfs 目录↔结构体字段对照树]]

---

## 0. 环境与工程结构

### 0.1 编模块要"借内核的构建系统"

```bash
uname -r                                  # 7.0.0-38-generic
ls -ld /lib/modules/$(uname -r)/build     # -> /usr/src/linux-headers-7.0.0-38-generic
```

**关键**：模块必须用**和运行内核完全一致**的编译选项 + `Module.symvers` + `vermagic`。
→ 所以用**发行版 headers**（`/lib/modules/$(uname -r)/build`），**不能**用 `/home/.../linux-source-code`（那是 7.3.0-rc6，编出来 `insmod` 会 `invalid module format`）。

### 0.2 kbuild 多目录：`Kbuild` 与 `Makefile` 分工

```
mydriver/
├── Makefile      ← 你敲 make 时读它（提供 all:/clean: 入口）
├── Kbuild        ← 内核读它（提供 obj-m 清单）
├── build.sh
├── 1/ 2/ 3/ 4/   ← 每一关一个子目录
│   ├── Kbuild    ← 子目录的 obj-m
│   └── *.c
```

| 你运行 | GNU make 读谁 | 干什么 |
| --- | --- | --- |
| `make`（裸跑） | **`Makefile`** | 找到 `all:` → 转调内核 |
| 内核处理 `M=$(PWD)` | **`Kbuild`**（优先于 `Makefile`） | 读 `obj-m` 决定编什么 |

```makefile
# Makefile（顶层）
KDIR ?= /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)
all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```
```makefile
# Kbuild（顶层）：进子目录
obj-m += 4/
```
```makefile
# 4/Kbuild：编这个文件
obj-m += hello4.o
```

**`obj-m` 的四种写法**：

| 写法 | 含义 |
| --- | --- |
| `obj-m += foo.o` | 编当前目录 `foo.c` → `foo.ko` |
| `obj-m += sub/` | 递归进 `sub/` 读它的 `Kbuild` |
| `obj-m += a.o b.o` | 编**两个独立**模块（`a.ko`、`b.ko`） |
| `foo-objs := a.o b.o` + `obj-m += foo.o` | **一个**模块由多个 `.o` 合成 |

> ⚠️ 两个子目录产出**同名**模块会冲突（`module xxx already loaded`）——所以每关用了不同名字（`hello1`/`hello2`/`hello3`/`hello4`）。

---

## 1. 第 0 关：裸模块

```c
#include <linux/module.h>
#include <linux/init.h>

static int __init hello_init(void)
{
	pr_info("hello: 我被加载了\n");   /* 输出到 dmesg，不是终端 */
	return 0;                        /* 非 0 = 加载失败 */
}

static void __exit hello_exit(void)
{
	pr_info("hello: 我要走了\n");
}

module_init(hello_init);   /* = device_initcall（level 6）→ .initcall6.init 段 */
module_exit(hello_exit);

MODULE_LICENSE("GPL");     /* 必须！见 §5-4 */
MODULE_AUTHOR("h-000-h");
```

**验证**：`insmod` → `dmesg` → `ls /sys/module/hello/` → `rmmod`。
`/sys/module/hello/` 里的 `coresize/initsize/initstate/taint/sections/notes/uevent` 全是内核**自动**建的。

> 第 0 关产物只有 2 个 `sections`（`__mcount_loc`、`__patchable_function_entries`）——**段的数量 ≈ 模块用了多少内核高级特性**（对比 `kvm` 的一大堆）。

---

## 2. 第 2 关：字符设备 + `/dev` 节点

### 2.1 三件套，缺一不可

| 调用 | 管哪层 | 缺了会怎样 |
| --- | --- | --- |
| `alloc_chrdev_region(&devt, 0, count, name)` | **号**（`major:minor`） | 后面无从谈起 |
| `cdev_add(&cdev, devt, count)` | **I/O 后端**（`dev_t → fops` 进 `cdev_map`） | 号占着没人认领 → `open()` **`-ENXIO`** |
| `device_create(class, parent, devt, drvdata, fmt, ...)` | **前端**（建 `device` + 发 uevent） | **`/dev/xxx` 不出现** |

**三者用同一个 `devt` 串起来** —— 这就是整条链的枢纽。

### 2.2 顺序：**建时后端先、拆时前端先**

```c
/* init：后端先就绪，节点再出现 */
alloc_chrdev_region();
cdev_add();              /* ← 必须在 device_create 前 */
class_create();
device_create();         /* ← 一旦执行，/dev 立刻可能被 open */

/* exit：严格反过来 */
device_destroy();        /* 先让节点消失，杜绝新的 open */
class_destroy();
cdev_del();
unregister_chrdev_region();
```

> 否则会出现"**节点可见但后端缺失**"的坏状态。

### 2.3 fops 骨架

```c
static ssize_t hello_read(struct file *filp, char __user *buf, size_t len, loff_t *off)
{
	static const char msg[] = "read from hello\n";   /* ★ 数组，不是指针！见 §5-5 */
	const size_t total = sizeof(msg) - 1;
	size_t n;

	if (*off >= total) return 0;                     /* 读完 → EOF */
	n = min_t(size_t, len, total - *off);
	if (copy_to_user(buf, msg + *off, n))
		return -EFAULT;                          /* 未拷贝字节数 ≠ 0 */
	*off += n;
	return n;
}
```

---

## 3. 第 3 关：多实例（一个 cdev 管 4 个次号）

```c
#define DEV_COUNT 4

struct hello_inst { const char *msg; int index; };
static const struct hello_inst hello_insts[DEV_COUNT] = {
	{ "I am hello0\n", 0 }, { "I am hello1\n", 1 },
	{ "I am hello2\n", 2 }, { "I am hello3\n", 3 },
};

static int hello_open(struct inode *inode, struct file *filp)
{
	int minor = iminor(inode);                  /* ★ 次号 = 第几个实例 */
	if (minor >= DEV_COUNT) return -ENODEV;
	filp->private_data = (void *)&hello_insts[minor];   /* ★ 算一次 */
	return 0;
}

static ssize_t hello_read(...)
{
	const struct hello_inst *inst = filp->private_data;  /* ★ 用多次 */
	...
}
```

```c
alloc_chrdev_region(&devt, 0, DEV_COUNT, "hello");   /* 要 4 个连续次号 */
cdev_add(&cdev, devt, DEV_COUNT);                    /* 一个 cdev 覆盖 4 个 */
for (i = 0; i < DEV_COUNT; i++)
	device_create(cls, NULL, MKDEV(MAJOR(devt), i), NULL, "hello%d", i);
```

**验证结果**（真机）：
```
crw------- 505,0  /dev/hello0
crw------- 505,1  /dev/hello1
crw------- 505,2  /dev/hello2
crw------- 505,3  /dev/hello3
       ↑↑↑ ↑
    同一主号 不同次号
```

> **4 个节点 = 同一个 cdev + 同一套 fops**，靠 `iminor()` 区分。这就是"**主号定驱动、次号定实例**"。

**`private_data` 的意义**：`.open` 只调 1 次、`.read` 可能调 N 次 → 把"找出本实例"放 open 一次。

---

## 4. 第 4 关：`platform_driver` + 假设备

### 4.1 核心转变

| | 第 2/3 关 | **第 4 关** |
| --- | --- | --- |
| 设备谁造 | **你自己** `device_create` | **总线**给你的 |
| 你的入口 | `module_init` 里直接干 | **`probe(pdev)`** |

### 4.2 代码骨架

```c
#include <linux/platform_device.h>
#define DEV_NAME "hello_platform"

static struct platform_device *fake_dev;

static int plat_probe(struct platform_device *pdev)   /* ★ 总线把设备交到这里 */
{
	pr_info("plat: probe! name=%s id=%d\n", pdev->name, pdev->id);
	return 0;
}

static void plat_remove(struct platform_device *pdev)  /* ★ 返回 void（内核近期改的） */
{
	pr_info("plat: remove!\n");
}

static struct platform_driver plat_driver = {
	.probe  = plat_probe,
	.remove = plat_remove,
	.driver = { .name = DEV_NAME },   /* ★ 匹配靠它（.owner 由宏自动填） */
};

static int __init plat_init(void)
{
	int ret;
	/* 顺序 A：先设备后驱动 → 路径 B（driver_attach 扫已有设备） */
	fake_dev = platform_device_register_simple(DEV_NAME, -1, NULL, 0);
	if (IS_ERR(fake_dev)) return PTR_ERR(fake_dev);

	ret = platform_driver_register(&plat_driver);   /* → probe */
	if (ret) { platform_device_unregister(fake_dev); return ret; }
	return 0;
}

static void __exit plat_exit(void)
{
	platform_device_unregister(fake_dev);
	platform_driver_unregister(&plat_driver);       /* ★ unregister！见 §5-10 */
}
```

**顺序 B**（先驱动后设备）走路径 A（`bus_probe_device` 找已有驱动）。**两条都试过，`probe` 都会在"后注册的那个"时刻触发。**

### 4.3 实机验证到的两条对称软链

```
/sys/devices/platform/hello_platform/driver
    -> ../../../bus/platform/drivers/hello_platform     ← 设备 → 驱动
/sys/bus/platform/drivers/hello_platform/hello_platform
    -> ../../../../devices/platform/hello_platform      ← 驱动 → 设备
```
（由 `driver_sysfs_add()`，`dd.c:483` 建立 —— 对应 [[struct platform_driver 与 probe 时序]] §9）

### 4.4 ★ 不用写代码就能触发 probe/remove：手动 `bind`/`unbind`

```bash
echo hello_platform | sudo tee /sys/bus/platform/drivers/hello_platform/unbind   # → remove
echo hello_platform | sudo tee /sys/bus/platform/drivers/hello_platform/bind     # → probe
```
**实测 dmesg**：
```
[499.616399] helloplat: remove! name=hello_platform
[531.972688] helloplat: 这个设备是内核给我的,不是我造的
```
这就是 [[struct platform_driver 与 probe 时序]] §8 里 `bus.c:264-270` 那条 unbind 路径。

---

## 5. ⚠️ 踩坑集（最值钱的一节）

| # | 现象 | 病根 | 教训 |
| --- | --- | --- | --- |
| 1 | `error: expected declaration specifiers before string constant` | `MPDULE_LICENSE` 拼错（M-**P**-DULE） | **宏名拼错 → 编译器不认识 → 报"莫名"错误**。查拼写去头文件 `grep` |
| 2 | `没有规则可制作目标"module"` | make 目标写成单数 | kbuild 目标是 **`modules`** |
| 3 | `/lib/module/...: 没有那个文件或目录` | 路径写成单数 | 是 **`/lib/modules`** |
| 4 | `ERROR: modpost: GPL-incompatible module ... uses GPL-only symbol 'class_create'` | `MODULE_LICENSE("DSB")` | `class_create`/`device_create`/`class_destroy`/`device_destroy` **全是 `EXPORT_SYMBOL_GPL`** → 必须 GPL 兼容（`"GPL"` / `"Dual BSD/GPL"`）。见 §6 |
| 5 | `cat /dev/hello` 只吐出 `read fr` | `static const char *msg` + `sizeof(msg)` = **8**（指针大小） | 用 **`static const char msg[]`**（数组）或 `strlen` |
| 6 | `/proc/devices` 残留 `hello`；反复加载泄漏多份 | `unregister_chrdev_region(devt, 1)` 但注册了 4 个 | **注册/注销的 `count` 必须配对** |
| 7 | `insmod: File exists`，但模块名/类名都没被占 | `device_create(cls, NULL, devt, "%s%hhu", name, i)` **少传 `drvdata`** → 参数**整体左移** → 4 个设备**全叫 "hello"** → 第 2 个撞名 `-EEXIST` | **可变参数函数传错参数，编译器不报警**（`drvdata` 是 `void*`，字符串能塞；`fmt` 非字面量，gcc 不做格式检查） |
| 8 | 错误路径"清理没生效" | `while (--i)`：i=1 时 `--i` = 0 → 循环体**一次都不进** | 用 **`while (i--)`**（先判断再自减） |
| 9 | 残留挡住下次加载 | 内核里残留的是**活节点** | **内核资源泄漏不可回收，只能重启** |
| 10 | 重复 `insmod` 报 `Device or resource busy` | `exit` 里把 `platform_driver_unregister` 写成了 `register` → **驱动注册泄漏** | `rmmod` 释放的是**模块**，不是你**注册进内核**的东西 |

### 5.1 两个最阴的坑，展开说

**① 可变参数左移（#7）** —— `device_create` 的签名是
```c
device_create(const struct class *class, struct device *parent,
	      dev_t devt, void *drvdata, const char *fmt, ...);
```
少传 `drvdata` → `fmt` 变成 `"hello"`（无格式符）→ 4 个设备同名 → `kobject_add_internal failed for hello with -EEXIST`。
**编译器完全不报错**，因为 `void*` 什么都能塞、`fmt` 又不是字面量。

**② `rmmod` 释放的是模块，不是注册项（#10）**
```
你的模块内存（rmmod 后释放）        内核 platform_bus_type 的 drivers_kset
┌──────────────────────┐          ┌────────────────────────────┐
│ static platform_driver│◄──悬空───│ kobject "hello_platform"    │
└──────────────────────┘          │  (kzalloc，【内核堆】，不动) │
      ↑ 已释放                     └────────────────────────────┘
```
`bus.c:747` `priv = kzalloc_obj(*priv)` —— `driver_private` 在**内核堆**，不随模块释放。
→ 下次 `platform_driver_register` → `driver_find()` → `kset_find_obj(drivers_kset, name)`（`bus.c:1418`）**按名字查到残留** → `return -EBUSY`。
→ 残留还是**悬空指针**（`priv->driver` 指向已释放内存），潜在 UAF。

### 5.2 "配对" vs "顺序"——两个不同类的 bug

| 错误类型 | 你写错什么 | 后果 |
| --- | --- | --- |
| **配对错** | 位置该写 `unregister`，写了 `register` | **泄漏** → 下次 EBUSY（就是 #10） |
| **顺序错** | 函数都对，正序拆了 | **半活状态/竞态** → `-ENXIO`、sysfs 错乱、偶发 UAF |

**准绳不是"顺序",是"依赖"**：
> 注册按"**依赖被满足**"的顺序；注销按"**依赖别人的先走**"的顺序。

- `device_destroy` **必须**先于 `cdev_del`（节点依赖 cdev）
- `class_destroy` **必须**后于 `device_destroy`（设备依赖 class）
- `platform_device` ↔ `platform_driver` **无依赖 → 顺序随意**

---

## 6. 附：`MODULE_LICENSE` 为什么卡得这么死

内核只认 **GPL 兼容**：

```c
/* include/linux/license.h:7 */
return (strcmp(license, "GPL") == 0
	|| strcmp(license, "GPL v2") == 0
	|| strcmp(license, "GPL and additional rights") == 0
	|| strcmp(license, "Dual BSD/GPL") == 0
	|| strcmp(license, "Dual MIT/GPL") == 0
	|| strcmp(license, "Dual MPL/GPL") == 0);
```

**反直觉点**：MIT/BSD/Apache **比 GPL 宽松**，恰恰**不是** GPL 兼容。
因为内核要的是 **copyleft 的传递性**（衍生作品也得 GPL），而"宽松"意味着**不承诺**这一点。
→ 内核把它一律归为 **"not compatible to GPLv2"（专有）**，taint `'P'`，并拒绝 `EXPORT_SYMBOL_GPL` 符号。

**想用 MIT/BSD 就写 `"Dual MIT/GPL"`** —— 关键字是 **`Dual`**（双许可，用户可选 GPL）。

---

## 7. 调试工具箱（真机常用）

```bash
sudo dmesg | tail -5                  # 驱动自己的 pr_info + 内核报错
lsmod | grep <name>                   # 是否已加载
ls /sys/module/<name>/                # 模块的 sysfs 投影（version/parameters/sections…）
cat /proc/devices | grep <name>       # ★ 字符设备号是否泄漏
ls -l /dev/<node>                     # 节点 + rdev(major,minor)
ls -l /sys/class/<cls>/               # class 视图（软链接）
ls -l /sys/devices/.../<dev>/         # 真身（dev/uevent/power/subsystem/driver）
modinfo <x>.ko                        # .ko 里的 MODULE_* 元信息
echo <dev> | sudo tee .../drivers/<d>/unbind   # ★ 手动解绑 → remove
echo <dev> | sudo tee .../drivers/<d>/bind     # ★ 手动绑定 → probe
cat /sys/kernel/debug/devices_deferred         # deferred 链表
```

**clangd/编辑器补全**：
```bash
python3 /usr/src/linux-headers-$(uname -r)/scripts/clang-tools/gen_compile_commands.py \
    -d "$PWD" -o "$PWD/compile_commands.json" "$PWD"
```
（扫 kbuild 的 `.xxx.o.cmd` 隐藏文件，把每次编译的完整命令抽成 `compile_commands.json`）

---

## 8. 五关递进（本笔记覆盖 1~4 关）

| 关 | 做什么 | 核心概念 |
| --- | --- | --- |
| 0/1 | 裸模块 | `module_init` = level 6、kbuild |
| 2 | 字符设备 + `/dev` | 三件套、**顺序**、`fops`、`copy_to_user` |
| 3 | 多实例 | **一个 cdev 管一段次号**、`iminor`、`private_data` |
| 4 | `platform_driver` + 假设备 | **设备由总线给**、`probe`/`remove`、两条路径、`bind`/`unbind` |
| **5** | **probe 里建字符设备** | **`platform_set_drvdata`**、`device_create(parent=&pdev->dev)` |
| 6 | `platform_get_resource` / `ioremap` | 拿硬件资源 |
| 7 | `devm_*` 改造 | 托管资源，自动逆序释放 |
| 8 | 中断 | `devm_request_irq` / threaded IRQ |
| 9 | 自定义 sysfs 属性 | `attribute_group` / `show`/`store` |

---

## 🔗 9. 相关笔记

- [[struct platform_driver 与 probe 时序]] — `really_probe` / 两条路径 / deferred
- [[file_operations 与字符设备结构精读]] — `cdev` / `kobj_map` / `copy_to_user`
- [[struct class 与设备自动发现]] — `/sys/class` 软链接 / glue dir
- [[sysfs 目录↔结构体字段对照树]] — 你 sysfs 里看到的一切
- [[struct device 精读]] — `devt` / `parent` / `driver`
- [[struct module 精读]] — `/sys/module/*` 那堆文件

---

## 📚 10. 参考资料

1. **本机实测** — 运行内核 `7.0.0-38-generic`（Ubuntu），`/usr/src/linux-headers-7.0.0-38-generic`。
2. **Linux Kernel 源码（本仓库 7.3.0-rc6）** — `include/linux/device/driver.h`、`include/linux/platform_device.h`、`drivers/base/{dd.c,bus.c,core.c,driver.c}`、`include/linux/license.h`、`Documentation/process/license-rules.rst`。

（注：踩坑集全部为本机真实报错；代码骨架以实际编译通过为准。）
