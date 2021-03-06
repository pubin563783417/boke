<!-- README.md -->

# iOS奔溃日志分析

2021-01-25

iOS下App如果奔溃了，可以通过奔溃日志文件来定位崩溃的问题。下面主要来分析日志文件各个字段代表的是什么含义。

首先来看一个奔溃的日志信息：

```
Incident Identifier: xxxxx
CrashReporter Key:   bb11a137ea80e04abbab9b4385c7991e0d299599
Hardware Model:      iPhone10,2
Process:             xxx [476]
Path:                /private/var/containers/Bundle/Application/36537326-E60E-4015-9528-BC108022EE4A/xxx
Identifier:          com.xx.xx
Version:             6.8.0.0.1 (6.8.0)
Code Type:           ARM-64 (Native)
Role:                Foreground
Parent Process:      launchd [1]
Coalition:           com.xx.xxx [595]


Date/Time:           2017-11-13 14:12:34.2318 +0800
Launch Time:         2017-11-13 13:40:34.4590 +0800
OS Version:          iPhone OS 11.0.1 (15A403)
Baseband Version:    1.00.03
Report Version:      104

Exception Type:  EXC_BREAKPOINT (SIGTRAP)
Exception Codes: 0x0000000000000001, 0x000000018698639c
Termination Signal: Trace/BPT trap: 5
Termination Reason: Namespace SIGNAL, Code 0x5
Terminating Process: exc handler [0]
Triggered by Thread:  48

Application Specific Information:
Attempting to mutate immutable collection!

Filtered syslog:
None found
```

## 崩溃文件说明
- Incident Identifier : 崩溃报告的唯一标识符
- CrashReporter Key: 是与设备标识相对应的唯一键值。
- Hardware Model: 手机的设备类型
- Process: 应用名称。中括号里面的数字是闪退时应用的进程ID。
- Identifier： 应用唯一标识.(Bundle ID).
- Version : App版本号
- Code Type ：崩溃日志所在设备的架构. 会是ARM-64, ARM, x86-64, or x86中的一个.
- OS Version ：手机的系统版本号。
- Exception Type : 异常类型。可以为EXC_BAD_ACCESS EXC_BAD_INSTRUCTION EXC_ARITHMETIC等。
- Exception Codes : 处理器的具体信息有关的异常编码成一个或多个64位进制数。通常情况下，这个区域不会被呈现，因为将异常代码解析成人们可以看懂的描述是在其它区域进行的
- Termination Reason: 当一个进程被终止的时的原因。
- Triggered by Thread: 异常发生的线程。0为主线程，其他为子线程。

## 异常类型说明

### EXC_BAD_ACCESS

EXC_BAD_ACCESS(signal) – Bad Memory Access.是最常碰到的Crash，通常用于访问了不该访问的内存导致。

#### signal:

- SIGSEGV: 通常由于重复释放对象导致，这种类型在切换了ARC以后应该已经很少见到了。
- SIGABRT: 收到Abort信号退出，通常Foundation库中的容器为了保护状态正常会做一些检测，例如插入nil到数组中等会遇到此类错误。
- SIGSEGV:（Segmentation Violation），代表无效内存地址，比如空指针，未初始化指针，栈溢出等；
- SIGBUS：总线错误，与 SIGSEGV 不同的是，SIGSEGV 访问的是无效地址，而 SIGBUS 访问的是有效地址，但总线访问异常(如地址对齐问题)
- SIGILL：尝试执行非法的指令，可能不被识别或者没有权限
- SIGALRM : 程序超时信号
- SIGFPE : 程序浮点异常信号
- SIGHUP : 程序终端中止信号
- SIGINT : 程序键盘中断信号
- SIGKILL : 程序结束接收中止信号
- SIGTERM : 程序kill中止信号
- SIGSTOP : 程序键盘中止信号


### EXC_BAD_INSTRUCTION

进程试图执行非法或未定义指令。这个进程可能试图通过一个配置错误的函数指针，跳到一个无效的地址。

### EXC_ARITHMETIC

除零错误会抛出此类异常

### EXC_BREAKPOINT

类似于异常退出,这个异常是为了给附加的调试器中断的过程的机会在其执行一个特定的点。您可以从您自己的代码引发此异常使用__builtin_trap()函数。如果没有调试器连接,进程将被终止并生成崩溃报告。

### EXC_RESOURCE

这个进程超出了资源消耗的限制。这是一个从操作系统通知,进程是使用太多的资源。这虽然不是崩溃但也会生成崩溃日志。

### Exception Codes

常见的Exception Codes代码有以下几种

- 0x8badf00d错误码：Watchdog超时，意为“ate bad food”。
- 0xdeadfa11错误码：用户强制退出，意为“dead fall”。
- 0xbaaaaaad错误码：用户按住Home键和音量键，获取当前内存状态，不代表崩溃。
- 0xbad22222错误码：VoIP应用（因为太频繁？）被iOS干掉。
- 0xc00010ff错误码：因为太烫了被干掉，意为“cool off”。
- 0xdead10cc错误码：因为在后台时仍然占据系统资源（比如通讯录）被干掉，意为“dead lock”

## 参考地址

- [Exception Types in iOS crash logs](https://stackoverflow.com/questions/7446655/exception-types-in-ios-crash-logs)


