# Renode 运行与监控命令

> 本文命令均依据 Renode 官方文档（readthedocs）核实。
> 关联：[[renode外设仿真]] ｜ [[renode调试与仿真]] ｜ [[renode-stm32f4外设和使用]]

## 一、启动 Renode 的几种模式

默认在图形界面运行：会弹出一个**日志窗口**和**监控窗口（Monitor）**。

```bash
renode                # 图形模式（默认）
renode --console      # 控制台模式：Monitor 与日志打在同一个终端里
renode --disable-gui  # 无头模式：不创建图形窗口，可在 CI 中使用
renode -P 1234        # telnet 模式：把 Monitor 暴露到网络端口，日志窗口仍在本地
```

- 无头模式下 UART 分析器不再弹窗，而是默认输出到日志。
- telnet/无头模式可配合使用：`renode --console --disable-gui`。
- 退出：在 Monitor 中输入 `quit`。

## 二、Monitor 常用技巧

```text
help                    # 列出所有内置命令
help <命令>             # 查看某个命令的用法
Tab                      # 一次补全命令 / 两次列出所有候选
↑ ↓                     # 命令历史
Ctrl+R                  # 交互式搜索历史
Ctrl+Shift+V            # 粘贴
Ctrl+C                   # 清空当前输入
@<路径>                 # 命令中的文件路径用 @ 开头，可 Tab 补全
```

常用命令都有单字母别名，比如 `start` 可写成 `s`。

## 三、机器（Machine）的生命周期

Renode 里一个"虚拟开发板"就是一个 **machine**。所有机器挂在同一个仿真品台下。

```text
mach create                    # 创建空机器，自动命名为 machine-0、machine-1 …
mach create "my-board"         # 自定义名字
mach set "my-board"            # 切换 Monitor 上下文到指定机器（也可用下标 mach set 0）
mach list                      # 列出全部机器
help mach                      # 查看 mach 相关全部子命令
```

新建的机器只有一条系统总线 `sysbus`，没有内存和 CPU，还不能跑程序。
要查看当前机器有哪些外设：

```text
peripherals                    # 列出全部外设（带地址范围树）
peripherals uart               # 按名字模糊过滤
peripherals STM32_Timer        # 按模型名过滤
peripherals <0x40000000, 0x7FFFFFFF>   # 按地址范围过滤
```

## 四、加载平台（repl 外设描述文件）

平台用 `.repl` 文件描述。Renode 自带的平台放在安装目录的 `platforms/` 下。

```text
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery.repl   # 整块开发板
machine LoadPlatformDescription @platforms/cpus/stm32f4.repl               # 只有 CPU+内核外设
machine LoadPlatformDescription @./my_board.repl                           # 自己的接口板文件
```

加载后可用 `peripherals` 查看，得到一棵外设树（`sysbus / cpu / uart …`）。

## 五、访问外设（属性 / 行为）

在机器上下文里，用 `sysbus.<外设名>` 引用外设，可以读属性、调行为：

```text
sysbus.ddr Size                     # get 属性
sysbus.ddr Size 0x20000             # set 属性
sysbus.ddr ZeroAll                  # 调用行为（方法）
sysbus.ddr                          # 列出该外设所有可用属性/方法及用法
```

输入外设名不跟参数，会打印它全部可用的属性和方法，非常有用。

## 六、加载固件（ELF / BIN / HEX）

```text
sysbus LoadELF @build/firmware.elf        # 本地文件
sysbus LoadELF @https://xxx/fw.elf        # 也可以从 https 加载
sysbus LoadBinary @app.bin                # 原始二进制
sysbus LoadHEX  @app.hex                  # HEX 文件
sysbus LoadUImage @uimage                 # uImage 格式
```

- 多个 ELF 依次加载时，以最后加载的一个来决定入口 PC（Cortex-M 上即向量表偏移）。
- 想手工指定入口：`sysbus.cpu PC 0x08000200`，Cortex-M 用 `sysbus.cpu VectorTableOffset 0x08000200`。

## 七、运行 / 暂停 / 复位

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

## 八、.resc 脚本批量执行

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

## 九、宏（macro）与变量

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

## 十、外设挂载与插件

在机器加载的平台之外，还可以单独挂载其它 `.repl` 外设文件：

```text
machine LoadPlatformDescription @./emulation_peripheral.repl
```

GPIO 按键、LED 等常用外设的用法见 [[renode外设仿真]]。