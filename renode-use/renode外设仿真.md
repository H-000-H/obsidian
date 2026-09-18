# Renode 外设仿真（UART / GPIO / LED / 按键 / 网络 / CAN）

> 命令依 Renode 官方文档核实。
> 关联：[[renode用法]] ｜ [[renode调试与仿真]]

## 一、UART 串口仿真

### 1. 弹出窗口直接看（Analyzer）

```text
showAnalyzer sysbus.usart2
```

会弹出一个模拟串口终端窗口，直接看打印、可用键盘输入字符发给单片机。

### 2. 通过 TCP 端口对接外部调试助手（跨平台，最常用）

```text
emulation CreateServerSocketTerminal 1234 "term_usart2"   # 监听 1234 端口
connector Connect sysbus.usart2 term_usart2               # 把模拟串口连上去
```

之后用任意 TCP 客户端连 `localhost:1234`（如 PuTTY、串口助手、`nc`）即可收发。
参数最后的 `false` 可关闭握手数据（默认会发一批 telnet IAC 配置字节）：

```text
emulation CreateServerSocketTerminal 1234 "term_usart2" false
```

### 3. 输出重定向到文件

```text
sysbus.usart2 CreateFileBackend @uart_log.txt       # 打印内容写入文件
sysbus.usart2 CreateFileBackend @uart_log.txt true  # 第 2 参 true=立即刷新
sysbus.usart2 CloseFileBackend @uart_log.txt        # 停止
```

### 4. （Linux/macOS）伪终端 pty

```text
emulation CreateUartPtyTerminal "term" "/tmp/uart"
connector Connect sysbus.uart0 term
```

### 5. 在 Monitor 里直接接管串口

```text
uart_connect sysbus.usart2    # 后续键盘输入全部发给串口，按 ESC 退回 Monitor
```

## 二、GPIO 外设（LED / 按键 / 普通 IO）

常把 LED、按键做成挂在 GPIO 口上的"子外设"，需要在 `.repl` 文件里描述，再加载或并入平台。

### 1. LED

```text
; my_led.repl
Led1 : Miscellaneous.LED @ gpioPortC

gpioPortC:
    13 -> Led1@0      ; 把 GPIO C13 接到 Led1 的通道 0
logLevel -1 gpioPortC.Led1   ; 打开该 LED 的 NOISY 日志，看状态变化
```

启动后观察：

```text
machine LoadPlatformDescription @./my_led.repl   ; 单独挂载
showAnalyzer sysbus.gpioPortC                    ; 打开 GPIO 分析器
```

程序把 C13 脚拉高/拉低时，`showAnalyzer` 弹窗里会看到对应颜色变化。

### 2. 按键（Button）

```text
; my_button.repl
Button1 : Miscellaneous.Button @ gpioPortA
    invert: true            ; 低电平按下时为"按下"，否则反转
    -> gpioPortA@6

Button2 : Miscellaneous.Button @ gpioPortA
    invert: true
    -> gpioPortA@7
```

在 Monitor 里模拟按下/松开：

```text
sysbus.gpioPortA.Button1 Press            ; 按下
sysbus.gpioPortA.Button1 Release          ; 松开
sysbus.gpioPortA.Button1 PressAndRelease  ; 短按（按下并立刻松开）
```

用 `Button` 的 `ButtonState` 属性也能查询/设置：

```text
sysbus.gpioPortA.Button1 ButtonState      ; 查询
```

### 3. 普通 GPIO 电平读写

直接操作 GPIO 端口的引脚电平：

```text
sysbus.gpioPortA ReadPin 3
sysbus.gpioPortA WritePin 3 true
```

`showAnalyzer sysbus.gpioPortA` 可弹出引脚电平可视化窗口，适合观察波形/时序实验。

## 三、网络外设

### 1. 多机 / 板间组网（Switch + connector）

先创建一个以太网交换机（默认是暂停态，需启动）：

```text
emulation CreateSwitch "switch1"
connector Connect sysbus.ethernet switch1    # 把本机网卡挂上交换机（需切到对应机器上下文）
switch1 Start                                # 已有仿真运行时手工启动交换机
```

`connector Disconnect sysbus.ethernet switch1` 断开，`connector DisconnectFromAll sysbus.ethernet` 断开全部。
混收模式用 `switch1 EnablePromiscuousMode sysbus.ethernet` / `DisablePromiscuousMode`。

> 说明：联网接入真实主机（把模拟网卡桥到本机虚拟网卡）、Wireshark 抓包等，涉及创建网卡背板/桥接命令，详见官方 networking / host-integration 章节，不同版本命令差异较大，使用时先 `help emulation` 确认。

## 四、CAN / SPI / I2C 外设

Renode 提供这些总线外设模型（如 `STMCAN`、`STM32SPI`、`STM32F4_I2C`），加载平台后按名引用即可：

```text
analyzers sysbus.can1              # 查看可用分析器
showAnalyzer sysbus.can1           # 弹窗显示 CAN 收发
```

### 两片 MCU 之间连 CAN（CAN Hub）

```text
emulation CreateCANHub "can_hub"             # 建 CAN 集线器（同 UART Hub）
# 切到机器 0：
connector Connect sysbus.can1 can_hub
# 切到机器 1：
connector Connect sysbus.can1 can_hub
can_hub Start                                # Hub 默认暂停，需启动（随整车仿真 start 也会自动启动）
```

### 板机对接真实 CAN 网络（仅 Linux）

```text
machine CreateSocketCANBridge "socketcan" "vcan1" ensureFdFrames=true
connector Connect sysbus.can1 socketcan
```

## 五、多机之间用 UART / GPIO 相连

### UART Hub（对称连接，双向）

```text
emulation CreateUARTHub "uartHub"
# 机器 0：
connector Connect sysbus.uart uartHub
# 机器 1：
connector Connect sysbus.uart uartHub
uartHub StrictMode True             # 可选：波特率/校验/停止位不一致时丢弃而非仅告警
```

### GPIOConnector（非对称、单向，需指定源/目标脚）

```text
emulation CreateGPIOConnector "gpio-con"
# 源机：
connector Connect sysbus.gpio gpio-con
gpio-con SelectSourcePin sysbus.gpio 7
# 目标机：
connector Connect sysbus.gpio gpio-con
gpio-con SelectDestinationPin sysbus.gpio 4
```

方向只能从 SourcePin 到 DestinationPin，要双向就再建一个方向相反的 connector。

## 六、一键查看某外设支持的分析器

不确定某外设能弹什么窗口时，先问：

```text
analyzers sysbus.usart2      # 列出可用分析器 id
analyzers default sysbus.usart2   # 默认分析器 id
```
```