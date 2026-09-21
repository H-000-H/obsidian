---
title: Renode 运行与监控命令完整手册（启动 / Monitor / 机器 / 平台 / 固件 / 脚本）
tags:
  - Renode
  - Simulation
  - Monitor
  - Embedded
  - Firmware
  - Script
status: Completed
created: 2026-09-20
---

# 🖥️ Renode 运行与监控命令完整手册（启动 / Monitor / 机器 / 平台 / 固件 / 脚本）

> [!INFO] 笔记概览
> - **核心命题**：Renode 的一切操作都通过 **Monitor**（命令控制台）完成；一次仿真的骨架是"**建机器 → 载平台 → 连外设 → 载固件 → start**"，这套流程可写进 `.resc` 脚本一键复现。
> - **内容范围**：四种启动模式 · Monitor 技巧 · 机器生命周期 · 加载 `.repl` 平台 · 访问外设 · 加载固件 · 运行/暂停/复位 · `.resc` 脚本 · 宏与变量
> - **推荐路径**：§1 启动 → §2 Monitor → §3 机器 → §4 平台 → §5 外设 → §6 固件 → §7 运行 → §8 脚本 → §9 宏
> - **命令约定**：命令依 Renode 官方文档（readthedocs）核实；`text` 为 Monitor 命令、`bash` 为宿主机 shell。

---

## 🗺️ 0. 一次仿真的骨架

```text
mach create "board"                       # ① 建机器
machine LoadPlatformDescription @...repl  # ② 载平台（CPU/内存/外设）
emulation CreateServerSocketTerminal ...  # ③ 连外设（串口/网络）
sysbus LoadELF @build/firmware.elf        # ④ 载固件
start                                     # ⑤ 跑起来
```

---

## 🚀 1. 启动 Renode 的几种模式

默认在图形界面运行：弹出**日志窗口**和**监控窗口（Monitor）**。

```bash
renode                # 图形模式（默认）
renode --console      # 控制台模式：Monitor 与日志打在同一个终端里
renode --disable-gui  # 无头模式：不创建图形窗口，可在 CI 中使用
renode -P 1234        # telnet 模式：把 Monitor 暴露到网络端口，日志窗口仍在本地
```

- 无头模式下 UART 分析器不再弹窗，默认输出到日志。
- telnet / 无头模式可组合：`renode --console --disable-gui`。
- 退出：在 Monitor 中输入 `quit`。

> [!TIP] CI 首选无头模式
> 自动化测试用 `renode --disable-gui` 配合 `.resc` 脚本，无窗口、可后台、易纳入流水线。

---

## ⌨️ 2. Monitor 常用技巧

```text
help                    # 列出所有内置命令
help <命令>             # 查看某个命令的用法
Tab                     # 一次补全命令 / 两次列出所有候选
↑ ↓                     # 命令历史
Ctrl+R                  # 交互式搜索历史
Ctrl+Shift+V            # 粘贴
Ctrl+C                  # 清空当前输入
@<路径>                 # 命令中的文件路径用 @ 开头，可 Tab 补全
```

> [!NOTE] 单字母别名
> 常用命令都有单字母别名，例如 `start` 可写成 `s`、`pause` 写成 `p`。

---

## 🧱 3. 机器（Machine）的生命周期

Renode 里一个"虚拟开发板"就是一个 **machine**，所有机器挂在同一个仿真平台下。

```text
mach create                    # 创建空机器，自动命名 machine-0、machine-1 …
mach create "my-board"         # 自定义名字
mach set "my-board"            # 切换 Monitor 上下文到指定机器（也可用下标 mach set 0）
mach list                      # 列出全部机器
help mach                      # 查看 mach 相关全部子命令
```

新建的机器只有一条系统总线 `sysbus`，没有内存和 CPU，还不能跑程序。查看当前机器有哪些外设：

```text
peripherals                    # 列出全部外设（带地址范围树）
peripherals uart               # 按名字模糊过滤
peripherals STM32_Timer        # 按模型名过滤
peripherals <0x40000000, 0x7FFFFFFF>   # 按地址范围过滤
```

> [!NOTE] 上下文很关键
> 多机仿真时，几乎所有命令都作用于"当前机器"。操作前用 `mach set` 切到目标机器，否则会误操作到别的节点。

---

## 🗂️ 4. 加载平台（`.repl` 外设描述文件）

平台用 `.repl` 文件描述，Renode 自带的平台放在安装目录 `platforms/` 下。

```text
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery.repl   # 整块开发板
machine LoadPlatformDescription @platforms/cpus/stm32f4.repl               # 只有 CPU+内核外设
machine LoadPlatformDescription @./my_board.repl                           # 自己的接口板文件
```

加载后可用 `peripherals` 查看，得到一棵外设树（`sysbus / cpu / uart …`）。STM32F407 的完整外设清单见 [[renode-stm32f4外设和使用]]。

---

## 🎚️ 5. 访问外设（属性 / 行为）

在机器上下文里用 `sysbus.<外设名>` 引用外设，可读属性、调行为：

```text
sysbus.ddr Size                     # get 属性
sysbus.ddr Size 0x20000             # set 属性
sysbus.ddr ZeroAll                  # 调用行为（方法）
sysbus.ddr                          # 不带参数 = 列出该外设全部属性/方法及用法
```

> [!TIP] 忘记用法就问外设本身
> 输入外设名不跟参数，会打印它全部可用的属性和方法，比翻文档更快。

---

## 📦 6. 加载固件（ELF / BIN / HEX）

```text
sysbus LoadELF @build/firmware.elf        # 本地文件
sysbus LoadELF @https://xxx/fw.elf        # 也可以从 https 加载
sysbus LoadBinary @app.bin                # 原始二进制
sysbus LoadHEX  @app.hex                  # HEX 文件
sysbus LoadUImage @uimage                 # uImage 格式
```

- 多个 ELF 依次加载时，以**最后加载的一个**决定入口 PC（Cortex-M 上即向量表偏移）。
- 手工指定入口：`sysbus.cpu PC 0x08000200`；Cortex-M 用 `sysbus.cpu VectorTableOffset 0x08000200`。

---

## ▶️ 7. 运行 / 暂停 / 复位

```text
start / s                          # 启动仿真（默认处于暂停态）
pause / p                          # 暂停
machine Reset                      # 复位当前机器（会执行 .repl 里 reset 段落）
sysbus.cpu ExecutionMode SingleStepBlocking   # 进入单步模式
sysbus.cpu Step                    # 执行一条指令
sysbus.cpu Step false              # 非阻塞单步（虚拟时间会前进）
sysbus.cpu ExecutionMode Continuous            # 恢复正常运行
Clear                              # 清空整个仿真（机器、外设、固件全部删除）
```

> [!CAUTION] 单步会推进虚拟时间
> `Step false`（非阻塞单步）会让虚拟时间前进，可能触发被测系统的超时逻辑。需要"冻结时间"调试时用 GDB，详见 [[renode调试与仿真]]。

---

## 📜 8. `.resc` 脚本批量执行

`.resc` 文件里就是一条条 Monitor 命令，语法与交互环境完全一致。

```text
# 示例 my_stm32.resc
mach create "stm32-node"
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery.repl
emulation CreateServerSocketTerminal 1234 "term_usart2"
connector Connect sysbus.usart2 term_usart2
sysbus LoadELF @build/firmware.elf
start
```

在 Monitor 里加载脚本：

```text
include @my_stm32.resc       # 执行脚本内容
start @my_stm32.resc         # 加载并立即 start
```

在系统终端里直接跑：

```bash
renode my_stm32.resc
```

脚本还可以 `include` 其它脚本，方便拆分成公共部分。

---

## 🔧 9. 宏（macro）与变量

脚本里常配合宏和变量做复位逻辑：

```text
$bin = @build/firmware.elf          # 变量

macro reset
"""
    sysbus LoadELF $bin
"""
runMacro $reset
```

`machine Reset` 复位机器时，`.repl` 里定义的 `reset:` 段落也会被执行。

> [!NOTE] 变量赋值语法
> `$bin = @...` 是直接赋值；`$bin ?= @...` 是"为空才赋值"（常用于允许外部覆盖默认固件路径）。`$CWD` 指 Renode 启动时的工作目录。

---

## 🔌 10. 外设挂载与插件

在机器加载的平台之外，还可单独挂载其它 `.repl` 外设文件：

```text
machine LoadPlatformDescription @./emulation_peripheral.repl
```

GPIO 按键、LED 等常用外设的用法见 [[renode外设仿真]]。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 跑通一次仿真前逐条核对
> - [ ] **启动模式**：交互用图形/控制台，CI 用 `--disable-gui`
> - [ ] **机器上下文**：多机时先 `mach set` 切到目标机器
> - [ ] **平台加载**：`LoadPlatformDescription` 后 `peripherals` 确认外设齐全
> - [ ] **外设连接**：串口/网络先 `Create...Terminal` 再 `connector Connect`
> - [ ] **固件入口**：多 ELF 时注意最后一个决定入口 PC，必要时手设 `VectorTableOffset`
> - [ ] **默认暂停**：建机后需 `start` 才运行
> - [ ] **脚本复现**：把整套流程写进 `.resc`，`renode xxx.resc` 一键复现
> - [ ] **单步陷阱**：`Step false` 会推进虚拟时间，调试优先用 GDB

---

## 🔗 思考与延伸索引

- [[renode外设仿真]] — UART / GPIO / LED / 按键 / CAN 等外设的仿真与可视化
- [[renode调试与仿真]] — GDB 调试、日志分级、快照与反向执行
- [[renode-stm32f4外设和使用]] — STM32F407 平台完整外设清单与地址映射
- [[UART-code]] — 被仿真固件里的 UART 驱动实现

---

## 📚 参考资料

1. **Renode 官方文档** — *Running Renode* / *Monitor* / *Machine Management*（readthedocs）.
2. **Renode 官方文档** — *Loading platforms & firmware* / *Scripts (.resc)* / *Macros & variables*.
3. **Renode 仓库** — `scripts/single-node/*.resc`、`platforms/`（平台与脚本样例）.
4. 项目内部笔记：[[renode外设仿真]]、[[renode调试与仿真]]。
