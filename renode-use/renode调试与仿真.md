---
title: Renode 调试与仿真完整手册（GDB / 单步 / 日志 / 快照 / 反向执行）
tags:
  - Renode
  - Simulation
  - Debug
  - GDB
  - Embedded
  - VSCode
status: Completed
created: 2026-09-20
---

# 🐞 Renode 调试与仿真完整手册（GDB / 单步 / 日志 / 快照 / 反向执行）

> [!INFO] 笔记概览
> - **核心命题**：Renode 把"整块板子"变成可暂停、可快照、可回退的软件对象；**CPU 被 GDB 暂停时虚拟时间不前进**，因此调试对模拟系统是"透明"的，不会触发超时误判。
> - **内容范围**：GDB Server 搭建与连接 · VSCode 一键单步 · 指令级单步与寄存器 · 日志分级/外设访问/函数轨迹 · 快照保存恢复 · 反向执行 · 典型调试流程
> - **推荐路径**：§1 GDB → §2 单步 → §3 日志 → §4 快照 → §5 反向执行 → §6 日常流程
> - **命令约定**：所有命令依 Renode 官方文档（readthedocs）核实；代码块中 `text` 为 Monitor 命令、`bash` 为宿主机 shell、`json` 为 VSCode 配置。

---

## 🗺️ 0. 调试能力总览

```text
GDB Server ──► 断点 / 单步 / 变量监视（与真实硬件体验一致）
指令级单步 ──► 不借助 GDB，在 Monitor 里直接 Step
日志系统   ──► 分级日志 / 外设读写记录 / 函数调用轨迹 / 静音未映射告警
快照存取   ──► Save / Load 整机状态，跨环境复现
反向执行   ──► reverse-step / reverse-continue（单核）
```

> [!TIP] 最推荐 GDB
> 断点、单步、看变量与真实硬件无异，且暂停不推进虚拟时间——这是 Renode 相比"printf 大法"最大的调试优势。

---

## 🐛 1. 用 GDB 调试（最推荐）

### 1.1 在 Renode 里开 GDB Server

```text
machine StartGdbServer 3333            # 在 3333 端口起 GDB Server
machine StartGdbServer 3333 true       # 第 3 参 true = 连接后自动 start（最简场景）
```

第 3 参 `autostartEmulation` 为 `true` 时，GDB 一连上 Renode 就自动启动仿真，省去手工 `start`。

### 1.2 用 GDB 连接

```bash
arm-none-eabi-gdb build/firmware.elf
(gdb) target remote :3333
(gdb) b main
(gdb) c
```

若没用 autostart，需先让仿真跑起来，两种方式任选：

```text
# 方式 A：在 Renode Monitor 里
start
# 然后回到 GDB
(gdb) c
```

```text
# 方式 B：直接在 GDB 里给 Renode 发命令
(gdb) monitor start
(gdb) c
```

### 1.3 多核 / 异构 CPU 指定核

默认 `StartGdbServer` 只处理全机器同构 CPU；异构场景要指定核或簇：

```text
machine StartGdbServer 3333 true cpuCluster="cortex-r5f"
machine StartGdbServer 3333 true sysbus.cpu1          # 单独加某个核
```

### 1.4 在 VSCode 里一键单步（自动启停 Renode）

官方推荐一套 3 文件配置，按 F5 即可：启动 Renode 载固件 → 连 3333 → 打断点、单步（Step Over/Into/Out）。官方样例在 Renode 仓库 `tools/vscode_config/`。

**① `.vscode/launch.json`（cppdbg）**

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug in Renode",
            "type": "cppdbg",
            "request": "launch",
            "preLaunchTask": "Run Renode",
            "postDebugTask": "Close Renode",
            "miDebuggerServerAddress": "localhost:3333",
            "cwd": "${workspaceRoot}",
            "miDebuggerPath": "arm-none-eabi-gdb",
            "program": "${workspaceRoot}/build/firmware.elf"
        }
    ]
}
```

**② `.vscode/tasks.json`（先编译 → 再跑 Renode 脚本，并等待 GDB Server 就绪）**

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Run Renode",
            "type": "shell",
            "command": "renode",
            "args": ["${workspaceFolder}/platform.resc"],
            "isBackground": true,
            "problemMatcher": {
                "source": "Renode",
                "pattern": { "regexp": "" },
                "background": {
                    "activeOnStart": true,
                    "beginsPattern": "Renode, version .*",
                    "endsPattern": ".*GDB server with all CPUs started on port.*"
                }
            }
        },
        {
            "label": "Close Renode",
            "command": "echo ${input:terminate}",
            "type": "shell",
            "problemMatcher": []
        }
    ],
    "inputs": [
        {
            "id": "terminate",
            "type": "command",
            "command": "workbench.action.tasks.terminate",
            "args": "terminateAll"
        }
    ]
}
```

**③ `platform.resc`（Renode 脚本，最后一条必须是 `machine StartGdbServer`）**

```text
$bin ?= $CWD/build/firmware.elf
include @scripts/single-node/stm32f4.resc      # 或自己写建机+载平台+连串口
machine StartGdbServer 3333                     # GDB Server 必须在脚本最后一行
```

> [!NOTE] 配置要点
> - `preLaunchTask` 以 `isBackground: true` 运行，靠 `problemMatcher` 检测到 `GDB server ... started on port 3333` 才继续启动 GDB，从而"等 Renode 就绪"。
> - `$?=` 是"变量为空则赋值"；`$CWD` 指 Renode 启动时的工作目录。
> - F5 后 **Step Over / Into / Out**、`F9` 断点、监视变量与真实硬件一样；仿真 CPU 暂停时虚拟时间不前进，不会引入超时误判。
> - 不需要自动启停时，可只保留 launch.json，手动在 Monitor 里 `machine StartGdbServer 3333`，再 `target remote :3333` 连接单步。

---

## 👣 2. 单步与读寄存器

### 2.1 指令级单步（不借助 GDB）

```text
sysbus.cpu ExecutionMode SingleStepBlocking   # 进入单步
sysbus.cpu Step                               # 走一条指令
sysbus.cpu Step false                         # 非阻塞单步（虚拟时间会前进）
sysbus.cpu ExecutionMode Continuous           # 恢复正常运行
```

### 2.2 查看当前位置 / 反查函数名

```text
sysbus.cpu PC                         # 当前执行地址，例如 0xC01890A8
sysbus FindSymbolAt `sysbus.cpu PC`   # 反查出函数名
```

> [!TIP] 命令替换
> 反引号 `` ` `` 是命令替换：取 `sysbus.cpu PC` 的结果作为参数传给 `FindSymbolAt`。

---

## 📋 3. 日志系统（Logging）

### 3.1 日志等级（共 5 级）

| 值 | 级别 | 说明 |
| :---: | :---: | :--- |
| -1 | NOISY | 最详细，会拖慢仿真 |
| 0 | DEBUG | 调试细节 |
| 1 | INFO | 常规信息 |
| 2 | WARNING | 警告 |
| 3 | ERROR | 错误 |

默认记录 NOISY 之外的所有级别。

```text
logLevel -1                     # 全局调到 NOISY（注意会拖慢仿真）
logLevel -1 sysbus.uart         # 只对某个外设调 NOISY
logLevel                        # 查看当前各对象级别
logLevel 2 file                 # 单独设置"文件后端"的级别
```

### 3.2 日志写入文件

```text
logFile @renode.log            # 开日志文件，不关闭控制台输出
logLevel 1 file                # 文件后端提升到 INFO（控制台仍可保持低级别）
```

### 3.3 记录外设访问

```text
sysbus LogPeripheralAccess sysbus.uart       # 记录某个外设的读写
sysbus LogAllPeripheralsAccess true           # 记录所有外设访问
```

开启后 CPU 访问该外设会打印类似：`uart: ReadByte from 0x0 (TransmitData), returned 0x0.`

### 3.4 函数调用轨迹（Trace）

```text
sysbus.cpu LogFunctionNames true                      # 记录每次进入函数
sysbus.cpu LogFunctionNames true "dev kobject"        # 只记录指定前缀
sysbus.cpu LogFunctionNames true ["dev kobject"] true # 再加一个 true 去掉重复行
```

更轻量的替代（不精确、无过滤）：

```text
sysbus.cpu LogTranslationBlockFetch true
```

### 3.5 屏蔽"未映射地址"警告

平台描述不完整时常刷 `WriteDoubleWord to non existing peripheral at 0x…`。对确实无所谓的地址区间静音：

```text
sysbus SilenceRange <0x80000 0x1000>      # 屏蔽 [0x80000, 0x81000) 区间日志
```

> [!CAUTION] 别把 SilenceRange 当"消音耳塞"
> 未映射访问往往暴露 `.repl` 平台描述缺外设或地址写错。先确认该地址真的无关紧要，再静音；否则会把真正的 bug 一起藏掉。

---

## 💾 4. 状态保存与恢复（快照）

把整个仿真（机器、外设、固件、运行状态）存成文件，换环境/换人可直接还原，无需再传固件和配置：

```text
Save @statefile.save            # 保存
Load @statefile.save            # 恢复（会先清空当前仿真）
autoSave true "0.2"             # 每 0.2 虚拟秒自动快照
reverseExecMode true            # 开启反向执行（配合 GDB 的 reverse-step 等）
```

- 快照可 gzip 压缩后加载：`Load @statefile.save.gz`
- 加载后需手动恢复上下文和串口窗口：

```text
mach set 0
showAnalyzer sysbus.uart
```

> [!WARNING] 版本兼容性
> 不同 Renode 版本生成的快照可能不兼容。跨版本复现问题时，优先固定 Renode 版本，或改用 `.resc` 脚本 + 固件重新搭建。

---

## ⏪ 5. 反向执行（Reverse Execution，单核）

配合 `reverseExecMode true`，GDB 里可回退：

```text
(gdb) reverse-step      # 后退一条指令（rs）
(gdb) reverse-stepi     # 后退一条汇编（rsi）
(gdb) reverse-continue  # 回到上一个断点/快照（rc）
(gdb) rsi 5             # 一次回退 5 条
```

---

## 🔄 6. 典型日常调试流程

1. 写一个 `.resc`：建机器 → 加载平台 → 串口接到 TCP → 载固件 → `machine StartGdbServer 3333`。
2. `renode xxx.resc` 启动。
3. 需要看 UART：`showAnalyzer sysbus.usart2`，或用串口助手连 1234 端口。
4. 要打断点：VSCode / GDB 连 3333。
5. 排查崩溃 / 未映射访问：开 `logLevel -1` + `LogPeripheralAccess`。
6. 出问题想复现：关键点 `Save @snap.save`，重来时 `Load @snap.save`。
7. 结束：`quit`。

---

## ✅ 落地校验清单（Checklist）

> [!SUCCESS] 调试环境搭建前逐条核对
> - [ ] **GDB Server**：`machine StartGdbServer 3333` 放在 `.resc` 最后一行
> - [ ] **自动启动**：简单场景用 `StartGdbServer 3333 true`，省去手工 `start`
> - [ ] **多核指定**：异构 CPU 用 `cpuCluster=` 或 `sysbus.cpuN` 明确目标核
> - [ ] **VSCode 就绪等待**：`problemMatcher.endsPattern` 匹配 GDB server started 日志
> - [ ] **单步模式**：用完 `SingleStepBlocking` 记得切回 `Continuous`
> - [ ] **日志级别**：NOISY 会拖慢仿真，定位完及时调回
> - [ ] **未映射告警**：先核实 `.repl` 是否缺外设，再决定是否 `SilenceRange`
> - [ ] **快照复现**：关键点 `Save`，跨版本注意兼容性
> - [ ] **反向执行**：仅单核支持，需先 `reverseExecMode true`

---

## 🔗 思考与延伸索引

- [[renode用法]] — 启动模式、Monitor 命令、机器生命周期与 `.resc` 脚本基础
- [[renode外设仿真]] — UART / GPIO / LED / 按键 / CAN 等外设的仿真与可视化
- [[renode-stm32f4外设和使用]] — STM32F407 平台完整外设清单与地址映射
- [[UART-code]] — 被仿真的 UART 驱动本身（IDLE + DMA 接收状态机）

---

## 📚 参考资料

1. **Renode 官方文档** — *Debugging with GDB* / *Using Renode from VSCode*（readthedocs）.
2. **Renode 官方文档** — *Logging* / *Saving and restoring state* / *Reverse execution*.
3. **Renode 仓库** — `tools/vscode_config/`（VSCode 一键调试样例配置）.
4. 项目内部笔记：[[renode用法]]、[[renode外设仿真]]。
