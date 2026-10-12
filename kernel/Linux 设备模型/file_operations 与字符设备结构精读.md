---
title: file_operations 与字符设备结构精读
aliases:
  - struct file_operations
  - fops
  - cdev
  - file_operations
  - 文件操作
tags:
  - linux/kernel
  - vfs
  - char-device
  - file-operations
status: Completed
created: 2026-10-12
kernel: "7.3.0-rc6 (mainline)"
arch: generic
---

# 📂 `file_operations` 与字符设备结构精读

> [!INFO] 笔记概览
> - **定位**：这是**文件 I/O 层**——用户态 `open/read/write/ioctl` 最终落到哪。它与设备模型/模块的"**记录层**"（`platform_device`/`struct device`/`struct module`）是**两层**：记录层负责"设备/驱动怎么被登记、配对、管理"，I/O 层负责"数据怎么进出"。
> - **内容范围**：`file_operations`（fops）· `FOP_*` 标志 · `cdev` · `file`/`inode` 与调用链 · 与 `device_driver`/`platform_driver`（驱动侧操作）的对照。
> - **引用格式**：`文件:行号`（本仓库 Linux 7.3.0-rc6）。

相关笔记：[[struct device 精读]] · [[platform-resource-mfd 结构精读]] · [[kobject-sysfs-kernfs映射分层]]

---

## 🧭 1. 两层分工（先厘清）

| 层 | 代表 | 负责 |
| --- | --- | --- |
| 设备模型 / **记录层** | `struct device` / `platform_device` / `platform_driver` / `struct module` | 设备登记、资源、总线匹配、`probe`、PM、sysfs |
| 文件 I/O 层 | `cdev` + **`file_operations`** | 用户态 `open/read/write/ioctl` 怎么落到驱动 |

> [!IMPORTANT] 有 `/dev` 才有 fops，且多半另建
> platform 驱动若要暴露 `/dev/xxx`，是在 `probe()` 里**另行** `cdev_init`/`cdev_add`/`device_create`；很多平台驱动根本不建 `/dev`（如 GPIO/时钟/regulator），或把接口交给 tty/IIO/input 子系统。

---

## 🗂️ 2. `struct file_operations`（fops）

```c
/* include/linux/fs.h:1919 */
struct file_operations {
    struct module *owner;          // 归属模块（引用计数，防卸载时被调用）
    fop_flags_t fop_flags;         // 能力标志（见 §3）
    ...
} __randomize_layout;
```

按用途分组（同文件 `:1922-1962`）：

| 分组 | 回调 | 说明 |
| --- | --- | --- |
| **归属/标志** | `owner`、`fop_flags` | 模块归属 + 能力标志 |
| **定位** | `llseek` | 改 `file->f_pos` |
| **读写（老接口）** | `read`、`write` | `char __user *` 直拷贝 |
| **读写（新接口）** | `read_iter`、`write_iter` | 走 `iov_iter`（现代优先） |
| **异步 I/O** | `iopoll`、`uring_cmd`、`uring_cmd_iopoll` | io_uring / 轮询 |
| **目录** | `iterate_shared` | 目录迭代 |
| **轮询** | `poll` | `select/poll/epoll` |
| **控制** | `unlocked_ioctl`、`compat_ioctl` | ioctl（含 32 位兼容） |
| **映射** | `mmap`、`mmap_prepare`、`get_unmapped_area`、`mmap_capabilities` | 内存映射 |
| **生命周期** | `open`、`release`、`flush` | 打开/释放/关闭时刷 |
| **同步/锁** | `fsync`、`fasync`、`lock`、`flock`、`check_flags`、`setlease` | 同步与文件锁 |
| **其它** | `splice_write`/`splice_read`/`splice_eof`、`fallocate`、`copy_file_range`、`remap_file_range`、`fadvise`、`show_fdinfo` | 零拷贝、预分配、范围复制、fdinfo |

> [!NOTE] `owner` 与模块卸载
> `owner` 让 VFS 在进入 fops 前 `try_module_get()`——这正是 [[struct module 精读]] §5 里 `refcnt`/`holders` 变化的来源之一（文件打开期间模块不能被卸）。

---

## 🏳️ 3. `FOP_*` 能力标志（`fs.h:1966-1980`）

| 标志 | 含义 |
| --- | --- |
| `FOP_BUFFER_RASYNC` / `FOP_BUFFER_WASYNC` | 支持异步缓冲读 / 写 |
| `FOP_MMAP_SYNC` | 映射支持同步缺页 |
| `FOP_DIO_PARALLEL_WRITE` | 支持多线程非独占 O_DIRECT 写 |
| `FOP_HUGE_PAGES` | 含大页 |
| `FOP_UNSIGNED_OFFSET` | 把 `loff_t` 当无符号（如 `/dev/mem`） |
| `FOP_ASYNC_LOCK` | 支持异步锁回调 |
| `FOP_DONTCACHE` | 支持无缓存读写 |

---

## 🔌 4. `struct cdev`：把 fops 挂到设备号

```c
/* include/linux/cdev.h:14 */
struct cdev {
    struct kobject kobj;                 // 内嵌 kobject（可挂 sysfs / 生命周期）
    struct module *owner;                // 归属模块
    const struct file_operations *ops;   // ← 本字符设备的 fops
    struct list_head list;
    dev_t dev;                           // 起始设备号（主:次）
    unsigned int count;                  // 设备号个数
} __randomize_layout;
```

典型用法（`cdev.h:23-35`）：

```c
cdev_init(&my_cdev, &my_fops);       // 绑定 fops
my_cdev.owner = THIS_MODULE;
cdev_add(&my_cdev, dev, 1);          // 注册到设备号 → VFS 可按 inode 找到
// 配合 device_create() 造 /dev 节点
```

> [!TIP] `cdev` 是"桥梁"
> inode 打开时，VFS 用 `i_rdev`（设备号）经 `cdev_map` 找到对应 `cdev`，取出 `cdev->ops`（即 `file_operations`）挂到 `file->f_op`。内核里 `inode->i_fop` 最终指向它。

---

## 🔗 5. 调用链：从 `open()` 到驱动

```mermaid
flowchart LR
    U["用户态 open/read/write/ioctl"] --> VFS["VFS 层"]
    VFS --> INODE["inode (i_rdev 设备号)"]
    INODE --> CDEV["cdev (cdev_map 查号)"]
    CDEV --> FOP["file_operations (cdev->ops)"]
    FOP --> DRV["驱动回调 read/write/ioctl …"]
```

- `open("/dev/xxx")`：VFS 找 inode → 由 `i_rdev` 查 `cdev` → `file->f_op = cdev->ops` → 调 `f_op->open`。
- 之后 `read/write/ioctl/mmap` 直接落到 `file->f_op` 对应回调。

---

## 🆚 6. 对照：驱动侧的"操作"（`device_driver` / `platform_driver`）

fops 是"**文件**操作"；驱动器件的另一套"操作"在驱动对象上（`probe/remove/shutdown` 等），属于**记录层**：

```c
/* include/linux/platform_device.h:284 */
struct platform_driver {
    int  (*probe)(struct platform_device *);
    void (*remove)(struct platform_device *);
    void (*shutdown)(struct platform_device *);
    int  (*suspend)(struct platform_device *, pm_message_t);
    int  (*resume)(struct platform_device *);
    struct device_driver driver;
    const struct platform_device_id *id_table;
    bool prevent_deferred_probe;
    bool driver_managed_dma;
};
```

`device_driver`（`include/linux/device/driver.h:100`）是其内嵌基类：`name`/`bus`/`owner`/`probe_type`/`of_match_table`/`acpi_match_table`/`probe`/`sync_state`/`remove`/`shutdown`/`pm`/`groups`/`dev_groups`…

> [!NOTE] 两套"操作"别混
> - `file_operations` → 用户态 I/O（`read/write/ioctl`），**只有需要 `/dev` 的设备才有**；
> - `platform_driver`/`device_driver` 的 `probe/remove` → 设备生命周期（匹配/绑定/解绑/PM），**每个驱动都有**。

---

## 🔗 7. 相关笔记

- [[platform-resource-mfd 结构精读]] — 记录层：`platform_device`/`resource`/`mfd_cell`
- [[struct device 精读]] — `dev_t`/`devt` 与 `/dev` 节点的来源
- [[kobject-sysfs-kernfs映射分层]] — `/sys`(元数据) vs `/dev`(I/O) 两张脸
- [[struct module 精读]] — `owner` 撑起的模块引用计数

---

## 📚 8. 参考资料

1. **Linux Kernel 源码（本仓库 7.3.0-rc6）** — `include/linux/fs.h`（`struct file_operations` `:1919`、`FOP_*` `:1966-1980`、`struct inode_operations` `:1994`）.
2. **同上** — `include/linux/cdev.h`（`struct cdev` `:14`、`cdev_init/add/del` `:23-35`）、`include/linux/platform_device.h`（`struct platform_driver` `:284`）、`include/linux/device/driver.h`（`struct device_driver` `:100`）.

（注：本文为源码精读笔记，事实来源以本仓库逐行核实为准。）
