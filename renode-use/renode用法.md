## 创建项目

- 创建测试平台 
```
mach create "xxx"
```
## 绑定平台
- 内置的平台仓库在platforms下
- 使用方法:
1. 使用标准内置路径
```
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery.repl
```
2. 只使用cpu 使用 外设
```
machine LoadPlatformDescription @platforms/cpus/stm32f4.repl
```
## 使用resc脚本使用renode
```
# 1. 初始化机器与平台
mach create "stm32-node"
machine LoadPlatformDescription @platforms/boards/stm32f4_discovery.repl

# 2. 串口重定向：暴露 TCP 供调试助手连接，或弹窗监控
emulation CreateServerSocketTerminal 1234 "term_usart2"
connector Connect sysbus.usart2 term_usart2
# 也可同时弹出原生窗口方便观察：
# showAnalyzer sysbus.usart2

# 3. 加载编译生成的固件（支持绝对路径或相对 resc 脚本的路径）
sysbus LoadELF @build/firmware.elf

# 4. 自动开始运行
start
```
- 在终端直接输入唤起renode
```
renode run_sim.resc 
```
## 对接(vscode 或者其他软件)断点
- 在resc 里面追加gdb
```
# 开放 3333 端口作为 GDB Server
machine StartGdbServer 3333
```

- 配置launch.json
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Renode GDB Debug",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/build/firmware.elf",
            "miDebuggerServerAddress": "localhost:3333",
            "miDebuggerPath": "arm-none-eabi-gdb",
            "MIMode": "gdb",
            "setupCommands": [
                { "text": "target remote :3333" },
                { "text": "monitor start" },
                { "text": "b main" },
                { "text": "c" }
            ]
        }
    ]
}
```
## 对接外部串口调试器
- 使用tcp对接
```
# 1. 创建一个监听在 1234 端口的 ServerSocket 终端对象，命名为 term_usart2
emulation CreateServerSocketTerminal 1234 "term_usart2"

# 2. 将目标单片机外设串口连接到该终端
connector Connect sysbus.usart2 term_usart2
```
## renode的logLevel
- -1=Noisy
- 0=Debug
- 1=Info
- 2=Warning
- 3=Error
# 外设
## 外设挂载(挂载一个repl外设文件从同级目录开始找)
```
machine LoadPlatformDescription @./emulation_peripheral.repl
```
## renode button 外设
- 新建repl文件
```
Button1 : Miscellaneous.Button @ gpioPortA
    invert: true
    -> gpioPortA@6

Button2 : Miscellaneous.Button @ gpioPortA
    invert: true
    -> gpioPortA@7
```
- 在renode终端里面输入
```
sysbus.gpioPortA.Button1 Press
sysbus.gpioPortA.Button1 Release
sysbus.gpioPortA.Button1 PressAndRelease
```

## renode led外设
```
Led1 : Miscellaneous.LED @ gpioPortC

gpioPortC:
    13 -> Led1@0
logLevel -1 gpioPortC.Led1
```