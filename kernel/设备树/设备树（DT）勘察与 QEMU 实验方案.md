---
title: 设备树（DT）勘察与 QEMU 实验方案
aliases:
  - Device Tree 入门
  - DTS 语法
  - QEMU 玩设备树
tags:
  - linux/kernel
  - device-tree
  - arm
  - qemu
status: In-progress
kernel: "源码基线 7.3.0-rc6（/home/h-000-h/project/linux-source-code）"
arch: arm32 (i.MX6UL) 实验 / x86_64 主机
---

# 🌳 设备树（DT）勘察与 QEMU 实验方案

> [!INFO] 一句话结论
> **x86_64 主机天然没有设备树**（走 ACPI），DT 只能在 ARM/RISC-V 上跑 → 本机唯一正路是 **QEMU 里启动 ARM32 i.MX6UL**（环境已齐、零下载）。
> 关联：[[ARM zImage head.S 启动 学习笔记]]（DTB 的传递就在这条链上）· [[KERNEL SET_UP]]（`setup_arch` 解析 DT）· [[驱动实战 2：file_operations 补全 · 互斥锁消融实验 · sysfs 属性]]

---

## 1. 为什么本机用不上 DT（实测证据）

```bash
grep -E '^CONFIG_(ACPI|OF)=' /boot/config-7.0.0-38-generic
# CONFIG_ACPI=y        ← 有
# （没有 CONFIG_OF）   ← 没有设备树支持
```

| | x86_64 | ARM / RISC-V |
| --- | --- | --- |
| 设备描述来源 | **ACPI 表**（BIOS/UEFI） | **设备树 DTB**（bootloader 传入） |
| 内核展开 | acpi_device → platform_device | of_node → platform_device |
| 驱动匹配 | `acpi_match_table`（或 `PRP0001` 兜底） | **`of_match_table`**（compatible 字符串） |
| 用户态可见 | `/sys/firmware/acpi/` | **`/proc/device-tree/`** |

## 2. DT 语法四件套

1. **`compatible`**：匹配驱动的唯一钥匙（对应驱动 `of_match_table` 里的字符串，形如 `"厂商,型号"`）
2. **`reg` + `#address-cells`/`#size-cells`**：单元格个数由**父节点**决定。父节点写 `#address-cells=<2>`、`#size-cells=<2>` 时，`reg = <0x0 0x40000000 0x0 0x20000000>` 才是"地址占 2 格 + 长度占 2 格"
3. **`status`**：`"okay"` 才展开成设备，`"disabled"` 直接不生成
4. **展开路径**：`DTB` → 内核 unflatten → 每个 enabled 且带 compatible 的节点 → `of_platform` 建 `platform_device` → 与驱动 match → `probe()`

## 3. 本机资源实测

| 资源 | 状态 |
| --- | --- |
| `qemu-system-arm` / `qemu-system-aarch64` | ✅ 均已安装 |
| `dtc`（设备树编译器） | ✅ 已安装 |
| `arm-linux-gnueabihf-gcc-15` | ✅ ARM32 交叉工具链 |
| `aarch64-linux-gnu-gcc-15` | ✅ ARM64 交叉工具链 |
| `~/project/linux-source-code` | **7.3.0-rc6**（笔记基线）已配 ARM32、编过 dtbs；**缺 `zImage`** |
| `~/project/linux/imx6ull/` | 6.18.37 教学树 + `qume-start.sh`（`-M mcimx6ul-evk -dtb ...`）+ busybox rootfs（NFS/initrd 两种根） |

> [!WARNING] 版本一致性铁律（沿用 [[驱动实战：从裸模块到 platform_driver]] §0.1）
> - **主机**跑模块 → 只能用发行版 headers（7.0.0-38），否则 `invalid module format`
> - **QEMU** 跑模块 → 必须用**启动该 VM 的那棵树**编（本次 = 7.3.0-rc6）

## 4. 实验方案（ARM32 / i.MX6UL，零下载）

**Step 0 补编内核镜像**（当前只有 dtbs，没有 zImage）

```bash
cd ~/project/linux-source-code
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage dtbs -j20
```

**Step 1 新建自己的板级 DTS**：`arch/arm/boot/dts/nxp/imx/imx6ul-hello-dt.dts`

```dts
/dts-v1/;
#include "imx6ul-14x14-evk.dts"          /* 基底只有 14 行：include dtsi + model/compatible */

/ {
	model = "i.MX6UL EVK + hello-dt";

	&lcdif { status = "disabled"; };  /* QEMU 不模拟 LCDIF，替代原脚本里的 fdtput 补丁 */

	hello_dev {                       /* ★ 只带属性、不带 reg（避免 ioremap 到不存在的地址） */
		compatible  = "h000,hello-dt";/* 驱动就认这个字符串 */
		hello-id    = <0x1234>;
		hello-name  = "hello from dts";
		status      = "okay";
	};
};
```

**Step 2 注册进 Makefile**：`arch/arm/boot/dts/nxp/imx/Makefile` 第 355-356 行的 `dtb-$(CONFIG_SOC_IMX6UL)` 列表里加一行 `imx6ul-hello-dt.dtb \`

**Step 3 编 dtb**

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- imx6ul-hello-dt.dtb
```

**Step 4 启动 QEMU**：复用 `imx6ull/qume-start.sh` 的启动行，`-dtb` 换成新 dtb，`-kernel` 换成 7.3.0-rc6 的 zImage；rootfs 建议用 `initrd` 模式（省去 NFS 服务）

**Step 5 内核侧验证（关键三处）**

```bash
ls /proc/device-tree/hello_dev/             # 内核解出来的 DT 节点
cat /proc/device-tree/hello_dev/hello-name  # 注意值带结尾 NUL
ls /sys/bus/platform/devices/ | grep hello  # of_platform 展开出的设备
```

**Step 6 驱动侧（`of_match_table` + 属性读取）**

```c
static const struct of_device_id hello_dt_ids[] = {
	{ .compatible = "h000,hello-dt" },     /* ★ 与 DTS 完全一致 */
	{ }
};
MODULE_DEVICE_TABLE(of, hello_dt_ids);

static struct platform_driver hello_dt_driver = {
	.probe  = hello_dt_probe,              /* 里面对 pdev->dev.of_node 用 of_property_read_* */
	.remove = hello_dt_remove,
	.driver = { .name = "hello_dt", .of_match_table = hello_dt_ids },
};
```

`probe` 里读属性：`of_property_read_u32(node, "hello-id", &v)` / `of_property_read_string(node, "hello-name", &s)`，再 `dev_info(&pdev->dev, ...)` 打印，用 `dmesg` 验证。

**Step 7 模块编译与投放**（同一棵树编 → 丢进 rootfs → VM 里 `insmod`）

```bash
make -C ~/project/linux-source-code M=$PWD ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- modules
cp hello_dt.ko ~/project/linux/imx6ull/rootfs/      # NFS 根直接可见；initrd 模式需重新打包
```

## 5. 踩坑预警

- `compatible` 打错一个字 → 驱动加载成功但 **probe 不执行**（无报错，最容易迷惑人）
- `status = "disabled"` → 节点在 `/proc/device-tree` 看得到，但**不会生成 platform_device**
- 节点名带 `@地址` 就必须写 `reg`，且数字个数要匹配父节点的 `#address-cells`/`#size-cells`
- 别对 QEMU 里不存在的物理地址 `ioremap`（会 abort）；本课只**打印** resource 值
- `/proc/device-tree` 只在有 DT 的系统存在（x86 上没有这个目录）
