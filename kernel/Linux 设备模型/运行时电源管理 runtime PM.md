---
title: 运行时电源管理 runtime PM
aliases:
  - runtime PM
  - dev_pm_info
  - dev_pm_domain
  - dev_pm_ops
  - genpd
  - 运行时电源管理
  - /sys/devices/xxx/power
tags:
  - linux/kernel
  - driver-model
  - power-management
  - runtime-pm
  - genpd
status: Completed
created: 2026-10-13
kernel: "7.3.0-rc6 (mainline)"
arch: generic
---

# ⚡ 运行时电源管理（runtime PM）

> [!INFO] 笔记概览
> - **定位**：runtime PM 是系统**运行时**、**按单个设备**动态省电——设备空闲就自动挂起、用时再唤醒。你 `ls` 到的 `/sys/devices/<dev>/power/` 目录就是它的用户态窗口。
> - **内容范围**：`struct dev_pm_info`（`dev->power`）· `/sys/.../power/` 属性表 · `enum rpm_status` · `dev_pm_ops`（回调来源）· `dev_pm_domain`/genpd（共享电源域）· runtime PM vs system sleep。
> - **引用格式**：`文件:行号`（本仓库 Linux 7.3.0-rc6）。

相关笔记：[[struct device 精读]] · [[struct bus_type 精读]] · [[struct class 与设备自动发现]]

---

## 🧠 1. `struct dev_pm_info power;` —— 设备的"PM 大脑"

内嵌在 `struct device`（`include/linux/pm.h:666`），是这台设备的运行时 PM 状态机 + 计数器 + 定时器。`/sys/devices/<dev>/power/` 里每个文件，读写的都是 `dev->power.xxx` 字段：

| `/sys/.../power/` 文件 | 字段（`include/linux/pm.h`） |
| --- | --- |
| `runtime_status` | `dev->power.runtime_status`（`:718`，`enum rpm_status`） |
| `runtime_usage` | `dev->power.usage_count`（`:702`） |
| `runtime_active_kids` | `dev->power.child_count`（`:703`） |
| `control`（on/auto） | `dev->power.runtime_auto`（`:709`） |
| `autosuspend_delay_ms` | `dev->power.autosuspend_delay`（`:721`） |
| `runtime_active_time`/`runtime_suspended_time` | `dev->power.active_time`/`suspended_time`（`:723-724`） |

> [!IMPORTANT] `power/` 目录 = 开在 `dev->power` 上的窗
> `cat runtime_status` 就是读 `dev->power.runtime_status`；`echo on > control` 就是改 `dev->power.runtime_auto`。属性文件由 PM 核心挂上（`drivers/base/power/sysfs.c`）。

---

## 📂 2. `/sys/devices/<dev>/power/` 属性速查

| 文件 | 读写 | 含义 | 源码 |
|---|---|---|---|
| `control` | **RW** | `on`=强制常活、**关 runtime PM**；`auto`=正常 runtime PM | `sysfs.c:97-110` |
| `runtime_status` | RO | 当前状态（见 §3 枚举） | `sysfs.c:149-179` |
| `runtime_enabled` | RO | 该设备 runtime PM 是否启用 | `sysfs.c:569-585` |
| `autosuspend_delay_ms` | **RW** | 自动挂起延迟(ms)；`-1`=从不自动挂。仅驱动调过 `pm_runtime_use_autosuspend()` 才生效 | `sysfs.c:79-89,181-208` |
| `async` | RO | suspend/resume 是否可异步（0/1） | `sysfs.c:588` |
| `runtime_active_time`/`runtime_suspended_time` | RO | 累计 active/suspended 时长(ms) | `sysfs.c:147` |
| `runtime_usage` | RO | 使用计数；**>0 有人在用，挂不了** | `sysfs.c:552-558` |
| `runtime_active_kids` | RO | 活跃子设备数；**>0 父设备不能挂** | `sysfs.c:560-567` |

> [!NOTE] 为什么你全看到了
> `runtime_usage`/`runtime_active_kids`/`runtime_enabled` 仅 `CONFIG_PM_ADVANCED_DEBUG` 下出现，`async` 仅 `CONFIG_PM_SLEEP` 下——你这台都有，说明这些配置开了。

**排障常用**：

```bash
cat runtime_status              # active / suspended？
echo on  > control              # 强制常活（怀疑被省电挂起时验证）
echo auto > control             # 恢复 runtime PM
cat runtime_usage               # >0 说明被占用，挂不了
```

---

## 🔢 3. `enum rpm_status`：runtime 状态取值

```c
/* include/linux/pm.h:599 */
enum rpm_status {
	RPM_INVALID = -1,
	RPM_ACTIVE,     // 全速工作
	RPM_RESUMING,   // 正在唤醒
	RPM_SUSPENDED,  // 已挂起
	RPM_SUSPENDING, // 正在挂起
};
```

`cat runtime_status` 打印的就是这几个字符串（`sysfs.c:159-176`）。

---

## 🔧 4. 回调来源：`dev_pm_ops`

`dev->power` 只是**状态**；真正"挂起/唤醒时做什么"是**回调 `struct dev_pm_ops`**（`runtime_suspend`/`runtime_resume`/`suspend`/`resume`/…）。它可来自三处：

| 来源 | 字段 |
| --- | --- |
| 总线 | `struct bus_type.pm`（`bus.h:111`） |
| 类 | `struct class.pm`（`class.h:69`） |
| 驱动 | `struct device_driver.pm`（`driver.h:137`） |

运行时 PM 框架（`drivers/base/power/runtime.c`）就是拿"**状态 `dev->power`** + **回调 `dev_pm_ops`**"驱动挂起/唤醒。

---

## 🏭 5. `struct dev_pm_domain *pm_domain;` —— 共享电源域（genpd）

`struct device` 里还有个 `pm_domain` 指针，**非空时该设备的 PM 回调改由电源域提供**，而不是总线/驱动：

```c
/* include/linux/pm.h:751 */
struct dev_pm_domain {
	struct dev_pm_ops ops;                       // 该电源域的 PM 回调
	int (*start)(struct device *dev);
	void (*detach)(struct device *dev, bool power_off);
	...
};
```

- 这就是 **genpd（通用电源域）** 机制（`drivers/genpd/`、`include/linux/pm_domain.h`）。
- 用途：SoC 把一组设备圈进同一电源/时钟域，**挂起域就把域内所有设备一起挂**（统一关电/开电）；无此概念的设备（如多数 PCI）`pm_domain` 为 `NULL`，直接走 §4 的 `dev_pm_ops`。
- 观测：`/sys/kernel/debug/pm_genpd/pm_genpd_summary` 列出各电源域及下属设备、开关状态。

**三件套**：状态在 `dev->power`（`dev_pm_info`）、回调在 `dev_pm_ops`（bus/driver/class）、**可被 `dev->pm_domain` 覆盖**，由 runtime PM 框架串起来。

---

## 🆚 6. runtime PM vs system sleep（别混）

| | runtime PM（本篇） | system sleep |
| --- | --- | --- |
| 入口 | 每个设备下的 `/sys/devices/<dev>/power/` | 全局 `/sys/power/` |
| 粒度 | **单个设备**，运行时动态 | **整机** suspend/hibernate |
| 触发 | 设备空闲自动挂 / 使用时唤醒 | 用户/策略让整机睡眠 |
| 状态字段 | `dev->power.runtime_status` | `dev->power.power_state`/`is_suspended`（`CONFIG_PM_SLEEP`） |

目录名都带 power，但是两套机制。

---

## 🔗 7. 相关笔记

- [[struct device 精读]] — `dev->power`、`dev->pm_domain`、`dev_pm_ops` 字段所在
- [[struct bus_type 精读]] — `bus_type.pm`、`irq_get_affinity`
- [[struct class 与设备自动发现]] — `class.pm`、`/sys/.../power/` 的投影视角

---

## 📚 8. 参考资料

1. **Linux Kernel 源码（本仓库 7.3.0-rc6）** — `include/linux/pm.h`（`struct dev_pm_info` `:666-731`、`enum rpm_status` `:599-605`、`struct dev_pm_domain` `:751`）.
2. **同上** — `drivers/base/power/sysfs.c`（`control` `:97-110`、`runtime_status` `:149-179`、`runtime_usage/active_kids` `:552-567`、`runtime_attrs` `:646`）、`drivers/base/power/runtime.c`（框架）.

（注：本文为源码精读笔记，事实来源以本仓库逐行核实为准。）
