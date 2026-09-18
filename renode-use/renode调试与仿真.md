# Renode 调试与仿真（GDB / 单步 / 日志 / 状态存取）

> 命令依 Renode 官方文档核实。
> 关键差异：**CPU 被 GDB 暂停时，仿真虚拟时间不会前进**，所以调试对模拟系统是"透明"的，不会触发超时误判。
> 关联：[[renode用法]] ｜ [[renode外设仿真]]

## 一、用 GDB 调试（最推荐的调试方式）

### 1. 在 Renode 里开 GDB Server

```text
machine StartGdbServer 3333            # 在 3333 端口起 GDB Server
machine StartGdbServer 3333 true       # 第 3 参 true = 连接后自动 start（最简单场景）
```

第 3 参 `autostartEmulation` 为 `true` 时，GDB 一连上 Renode 会自动启动仿真，省去手工 start。

### 2. 用 GDB 连接

```bash
arm-none-eabi-gdb build/firmware.elf
(gdb) target remote :3333
(gdb) b main
(gdb) c
```

如果没用 autostart，需要先让仿真跑起来，两种方式任选：

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

### 3. 多核 / 异构 CPU 时指定核

默认 `StartGdbServer` 只处理全机器同构 CPU 的情况；异构场景要指定核或簇：

```text
machine StartGdbServer 3333 true cpuCluster="cortex-r5f"
machine StartGdbServer 3333 true sysbus.cpu1          # 单独加某个核
```

### 4. 在 VSCode 里直接单步调试（自动启停 Renode）

官方推荐一套 3 个文件配置，点一下 F5 就能：先启动 Renode 并载固件 → 连上 3333 → 用 VSCode 打断点、单步（Step Over/Into/Out）。官方样例在 Renode 仓库 `tools/vscode_config/`。

**① `.vscode/launch.json`**（cppdbg）

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug in Renode",
            "type": "cppdbg",
            "request": "launch",
            "preLaunchTask": "Run Renode",          // F5 前先把 Renode 拉起来
            "postDebugTask": "Close Renode",        // 结束调试后关掉 Renode（可删）
            "miDebuggerServerAddress": "localhost:3333",
            "cwd": "${workspaceRoot}",
            "miDebuggerPath": "arm-none-eabi-gdb",
            "program": "${workspaceRoot}/build/firmware.elf"
        }
    ]
}
```

**② `.vscode/tasks.json`**（后台任务：先编译 → 再跑 Renode 脚本，并等待 GDB Server 就绪）

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

**③ `platform.resc`**（Renode 脚本，最后一条必须是 `machine StartGdbServer`）

```text
$bin ?= $CWD/build/firmware.elf
include @scripts/single-node/stm32f4.resc      # 或自己写建机+载平台+连串口
machine StartGdbServer 3333                     # GDB Server 必须在脚本最后一行
```

要点：
- `preLaunchTask` 的任务以 `isBackground: true` 运行，靠 `problemMatcher` 检测到 `GDB server ... started on port 3333` 才继续启动 GDB，从而"等 Renode 就绪"。
- `$?=` 是"变量为空则赋值"的写法；`$CWD` 指 Renode 启动时的工作目录。
- F5 进入调试点 **Step Over / Into / Out**、`F9` 断点、监视变量，与真实硬件一样；仿真 CPU 暂停时虚拟时间不前进，调试过程不会引入超时误判。
- 若不需要自动启停，也可以只保留 launch.json，手动先在 Monitor 里 `machine StartGdbServer 3333`（或 resc 里加一行），再用 `target remote :3333` 连接即可单步。

## 二、单步与读寄存器

### 1. 指令级单步（不借助 GDB）

```text
sysbus.cpu ExecutionMode SingleStepBlocking   # 进入单步
sysbus.cpu Step                               # 走一条指令
sysbus.cpu Step false                         # 非阻塞单步
sysbus.cpu ExecutionMode Continuous           # 恢复正常
```

### 2. 查看当前位置 / 反查到函数名

```text
sysbus.cpu PC                     # 当前执行的地址，例如 0xC01890A8
sysbus FindSymbolAt `sysbus.cpu PC`   # 反查出函数名
```

反引号 `` ` `` 是命令替换：取 `sysbus.cpu PC` 的结果作为参数。

## 三、日志（Logging）

### 1. 日志等级（共 5 级）

| 值 | 级别 |
|----|------|
| -1 | NOISY（最详细） |
| 0  | DEBUG |
| 1  | INFO |
| 2  | WARNING |
| 3  | ERROR |

默认只记录 NOISY 之外的所有级别。

```text
logLevel -1                     # 全局调到 NOISY（注意会拖慢仿真）
logLevel -1 sysbus.uart         # 只对某个外设调 NOISY
logLevel                        # 查看当前各对象级别
logLevel 2 file                 # 单独设置"文件后端"的级别
```

### 2. 日志写入文件

```text
logFile @renode.log            # 开日志文件，不关闭控制台输出
logLevel 1 file                # 文件后端提升到 INFO（控制台仍可保持低级别）
```

### 3. 记录外设访问

```text
sysbus LogPeripheralAccess sysbus.uart       # 记录某个外设的读写
sysbus LogAllPeripheralsAccess true           # 记录所有外设访问
```

开启后 CPU 访问该外设会打印类似：`uart: ReadByte from 0x0 (TransmitData), returned 0x0.`

### 4. 函数调用轨迹（Trace）

```text
sysbus.cpu LogFunctionNames true                      # 记录每次进入函数
sysbus.cpu LogFunctionNames true "dev kobject"        # 只记录指定前缀
```
```text
sysbus.cpu LogFunctionNames true ["dev kobject"] true # 再加一个 true 去掉重复行
```

更轻量的替代（不精确、无过滤）：

```text
sysbus.cpu LogTranslationBlockFetch true
```

### 5. 屏蔽"未映射地址"警告

平台描述不完整时常刷 `WriteDoubleWord to non existing peripheral at 0x…`。对确实无所谓的地址区间静音：

```text
sysbus SilenceRange <0x80000 0x1000>      # 屏蔽 [0x80000, 0x81000) 区间日志
```

## 四、状态保存与恢复（快照）

把整个仿真（机器、外设、固件、运行状态）存成文件，换环境/换人可直接还原，无需再传固件和配置：

```text
Save @statefile.save            # 保存
Load @statefile.save            # 恢复（会先清空当前仿真）
autoSave true "0.2"             # 每 0.2 虚拟秒自动快照
reverseExecMode true            # 开启反向执行（配合 GDB 的 reverse-step 等）
```

- 文件可 gzip 压缩后加载：`Load @statefile.save.gz`
- 加载后需手动恢复上下文和串口窗口：

```text
mach set 0
showAnalyzer sysbus.uart
```

- 不同 Renode 版本生成的快照可能不兼容。

## 五、反向执行（Reverse Execution，单核）

配合 `reverseExecMode true`，GDB 里可回退：

```text
(gdb) reverse-step      # 后退一条指令（rs）
(gdb) reverse-stepi     # 后退一条汇编（rsi）
(gdb) reverse-continue  # 回到上一个断点/快照（rc）
(gdb) rsi 5             # 一次回退 5 条
```

## 六、典型日常调试流程

1. 写一个 `.resc`：建机器 → 加载平台 → 串口接到 TCP → 载固件 → `machine StartGdbServer 3333`。
2. `renode xxx.resc` 启动。
3. 需要看 UART：`showAnalyzer sysbus.usart2`，或用串口助手连 1234 端口。
4. 要打断点：VSCode / GDB 连 3333。
5. 排查崩溃/未映射访问：开 `logLevel -1` + `LogPeripheralAccess`。
6. 出问题想复现：关键点 `Save @snap.save`，重来时 `Load @snap.save`。
7. 结束：`quit`。