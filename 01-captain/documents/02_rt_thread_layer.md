# RT-Thread 层：内核、调度与系统服务

> 上一层：[应用层](./01_application_layer.md)  
> 下一层：[BSP 与软件移植层](./03_bsp_porting_layer.md)  
> 硬件层：[RTL/SoC 层](./04_rtl_soc_layer.md)

## 1. RT-Thread 在系统里负责什么

RT-Thread 是嵌入式实时操作系统。它管理“哪些线程可以运行、现在运行谁、线程等待什么、何时被唤醒”，并提供线程间同步和通信。

在本项目中，RT-Thread 运行在单核 RV32 CPU 的机器模式，通常具有以下特点：

- 没有 MMU；
- 没有用户进程隔离；
- 内核、驱动和应用共享同一个 32 位地址空间；
- 所有源码静态链接成同一个 `firmware.elf`；
- 线程各自拥有栈，但可以访问同一套全局变量和硬件地址；
- Timer IRQ 是调度时间基准；
- UART 是启动日志和 FinSH 的 console。

RT-Thread 不是负责“把 C 程序下载到板子”的软件。程序装载由构建工具、FPGA BRAM 初始化或 Bootloader 完成。

## 2. RT-Thread 的上下接口

RT-Thread 位于应用和 BSP 之间。

```text
应用层
  │ 线程、IPC、定时器、打印 API
  ▼
RT-Thread 内核
  │ context switch、interrupt、tick、board init
  ▼
BSP / RISC-V port
  │ CSR、MMIO、Trap
  ▼
CPU 与 SoC
```

### 2.1 向上提供给应用的接口

主要包括：

```c
rt_thread_init()
rt_thread_create()
rt_thread_startup()
rt_thread_mdelay()

rt_sem_init()
rt_sem_take()
rt_sem_release()

rt_mutex_init()
rt_mutex_take()
rt_mutex_release()

rt_mb_init()
rt_mb_send()
rt_mb_recv()

rt_mq_init()
rt_mq_send()
rt_mq_recv()

rt_timer_init()
rt_timer_start()
rt_tick_get()

rt_malloc()
rt_free()
rt_kprintf()
```

### 2.2 向下要求 BSP/CPU port 提供的接口

内核本身不知道 RISC-V 寄存器如何保存，也不知道 UART 在哪个地址。它依赖：

```c
rt_hw_stack_init()
rt_hw_context_switch()
rt_hw_context_switch_to()
rt_hw_context_switch_interrupt()

rt_hw_interrupt_disable()
rt_hw_interrupt_enable()

rt_hw_board_init()
rt_hw_console_output()
rt_hw_console_getchar()
```

此外，硬件 Timer ISR 必须周期性调用：

```c
rt_tick_increase();
```

这就是 RT-Thread 与 BSP 的主要边界。

## 3. RT-Thread 开源仓库的组成

建议固定官方稳定 tag，不直接跟随 `master`。截至本文编写时，官方最新稳定发布为 v5.2.2；去年的优秀作品使用较早的 RT-Thread Nano 3.1.5。

官方仓库中与本项目关系最密切的目录如下：

```text
rt-thread/
├─ src/                    # 内核实现
├─ include/                # 公共头文件
├─ libcpu/risc-v/common/   # RV32/RV64 通用移植层
├─ components/finsh/       # FinSH/MSH
├─ components/drivers/     # 可选设备框架
├─ bsp/                    # 官方板级参考
├─ Kconfig
└─ tools/
```

### 3.1 `src/`

常见文件职责：

| 文件 | 主要职责 |
|---|---|
| `thread.c` | 线程初始化、启动、挂起、恢复 |
| `scheduler_up.c` | 单核调度器 |
| `scheduler_comm.c` | 调度公共逻辑 |
| `clock.c` | Tick 和延时换算 |
| `timer.c` | 软件定时器 |
| `ipc.c` | 信号量、互斥量、事件等 |
| `object.c` | 内核对象管理 |
| `idle.c` | idle 线程 |
| `irq.c` | 中断嵌套计数和公共接口 |
| `mem.c`、`memheap.c` 等 | 不同内存管理器 |
| `kservice.c` | 字符串、打印等内核服务 |
| `components.c` | 组件初始化和 user main 启动 |

不要随意修改这些内核源码来绕过 BSP 问题。线程切换失败时，应先检查：

- Trap 保存帧；
- `mepc/mstatus`；
- 栈对齐；
- Timer IRQ；
- 编译器生成的 ISA；
- `rtconfig.h`。

### 3.2 `include/`

应用和 BSP 常用：

```text
rtthread.h
rthw.h
rtdef.h
rtcompiler.h
rtdbg.h
```

`rtthread.h` 是应用常用总入口；`rthw.h` 包含硬件移植接口声明。

### 3.3 `libcpu/risc-v/common/`

当前官方 common port 包含：

```text
context_gcc.S
interrupt_gcc.S
cpuport.c
cpuport.h
riscv-ops.h
rt_hw_stack_frame.h
trap_common.c
atomic_riscv.c
```

这些文件提供通用的 RISC-V 上下文结构，但仍假定硬件 CSR 和 Trap 行为正确。本项目必须逐项核对：

- RV32，而不是 RV64；
- `REGBYTES=4`；
- 不启用 RVC；
- 不启用 FPU 上下文；
- 不启用 SMP；
- `mscratch` 用法与中断栈一致；
- `mret` 返回到机器模式；
- 是否产生当前 CPU 不支持的 A 扩展指令。

## 4. 什么是线程

线程可以理解为“一套独立的执行现场”：

```text
线程控制块
├─ 当前状态
├─ 优先级
├─ 剩余时间片
├─ 栈指针
├─ 等待对象
└─ 定时信息

线程栈
├─ 函数调用现场
├─ 局部变量
└─ 被切换时保存的寄存器
```

CPU 一次只执行一个线程。调度器通过保存旧线程寄存器、切换栈指针、恢复新线程寄存器，让多个线程表现得像同时运行。

### 4.1 线程状态

常见状态：

| 状态 | 含义 |
|---|---|
| init | 已初始化，尚未启动 |
| ready | 可以运行，正在等待 CPU |
| running | 当前占用 CPU |
| suspend | 等待延时、IPC 或外部事件 |
| close | 线程退出，等待回收或已关闭 |

单核系统同一时刻只有一个 running 线程。

## 5. 调度器如何决定运行谁

RT-Thread 常用的是固定优先级抢占调度：

1. 每个优先级有 ready 队列；
2. 调度器选择最高优先级的 ready 线程；
3. 高优先级线程就绪后，可以抢占低优先级线程；
4. 同优先级线程可按时间片轮转。

假设数字越小优先级越高：

```text
线程 A：优先级 8，等待 UART 数据
线程 B：优先级 15，执行控制逻辑
线程 C：优先级 25，闪烁 LED
```

平时 A 阻塞，B 或 C 运行。UART 中断释放 A 等待的信号量后，A 成为 ready。中断退出前，内核发现 A 优先级最高，切换到 A。

## 6. 上下文切换有哪两种入口

### 6.1 线程主动引发的切换

例如：

```text
rt_thread_mdelay()
rt_sem_take() 且资源不可用
rt_thread_yield()
```

调用链概念上是：

```text
应用 API
  ↓
修改当前线程状态/ready 队列
  ↓
调度器选出目标线程
  ↓
rt_hw_context_switch()
  ↓
汇编保存旧 sp，加载新 sp
  ↓
恢复目标线程寄存器并继续运行
```

### 6.2 中断退出时发生的切换

例如 Timer ISR 唤醒高优先级线程：

```text
Timer IRQ
  ↓
保存完整中断现场
  ↓
rt_interrupt_enter()
  ↓
Timer ISR 调用 rt_tick_increase()
  ↓
内核决定需要切换
  ↓
rt_hw_context_switch_interrupt()
  ↓
中断退出时换成目标线程的栈
  ↓
mret
```

中断内不能像普通线程一样随意切换栈。RT-Thread common port 通过中断切换标志和 from/to thread 指针，在退出路径完成切换。

## 7. 系统 Tick 是什么

Tick 是 RT-Thread 的时间刻度。若：

```c
#define RT_TICK_PER_SECOND 1000
```

则每个 Tick 是 1 ms。

Tick 依赖硬件定时器：

```text
machine_timer.mtime 达到 mtimecmp
  ↓
irq_timer_i = 1
  ↓
CPU 进入 machine timer interrupt
  ↓
BSP Timer ISR 设置下一次 compare
  ↓
rt_tick_increase()
```

`rt_tick_increase()` 会：

- 增加系统 Tick；
- 更新线程延时；
- 检查软件定时器；
- 处理时间片；
- 使到期线程重新 ready；
- 必要时请求调度。

### 7.1 Tick 不工作时的典型现象

- 第一个线程能打印；
- 调用 `rt_thread_mdelay()` 后再也不返回；
- `rt_tick_get()` 一直不变；
- shell 可能启动后卡住；
- 软件定时器不执行；
- 同优先级时间片轮转失效。

这类问题应检查硬件 Timer、CSR 中断使能和 ISR，不应先修改线程 API。

## 8. 软件定时器和硬件定时器的区别

硬件定时器是 SoC RTL 外设，直接计数并产生 IRQ。

软件定时器是 RT-Thread 内核对象：

```c
rt_timer_init()
rt_timer_start()
```

软件定时器依赖系统 Tick，由内核判断何时调用回调。一个硬件 Tick 定时器可以支持许多软件定时器。

## 9. IPC 在内核内部做什么

### 9.1 信号量

信号量包含一个计数值和等待线程队列：

```text
计数 > 0：take 成功，计数减 1
计数 = 0：线程进入等待队列
release：计数增加或直接唤醒等待线程
```

### 9.2 互斥量

互斥量记录拥有者。常见实现还包含优先级继承，降低优先级反转风险。

### 9.3 邮箱和消息队列

邮箱传递机器字，消息队列复制固定大小消息。二者都可能使发送者或接收者阻塞。

IPC 的核心不是“保存数据”本身，而是把线程状态与调度器连接起来。

## 10. 内存管理

RT-Thread 支持多种内存管理方式。比赛第一版可选：

### 10.1 全静态

- 线程栈静态数组；
- 信号量和消息队列控制块静态定义；
- CoreMark 使用静态数据；
- 不启用动态 heap。

优点是最容易验证，适合内核 bring-up。

### 10.2 小型 heap

链接脚本提供：

```text
__heap_start
__heap_end
```

`rt_hw_board_init()` 调用：

```c
rt_system_heap_init((void *)__heap_start,
                    (void *)__heap_end);
```

之后才可安全使用：

```c
rt_thread_create()
rt_malloc()
```

如果 heap 边界覆盖 `.bss`、boot stack 或 IRQ stack，会出现随机崩溃。

## 11. idle 线程

系统没有其他 ready 线程时运行 idle：

```text
idle thread
├─ 清理已退出的动态线程
├─ 调用 idle hook
└─ 可执行低功耗等待
```

当前 CPU 没有 `WFI` 时，idle 可以执行普通空循环。不要在 idle hook 中调用会阻塞的 API。

## 12. FinSH/MSH 属于哪一部分

FinSH 是 RT-Thread 组件，不是调度器核心。MSH 是常用命令行模式。

它包含：

- shell 线程；
- console 输入；
- 行编辑；
- 命令解析；
- 命令符号表；
- `help/ps/free` 等内置命令。

数据路径：

```text
UART RX
  ↓
BSP console get-char 或 serial device
  ↓
FinSH shell 线程
  ↓
解析命令
  ↓
在 FSymTab 中查找函数
  ↓
调用应用命令
```

FinSH 依赖以下条件：

- UART TX/RX 可用；
- 至少一个 shell 线程栈；
- Tick 和调度正常；
- 链接脚本保留 `FSymTab/VSymTab`；
- 这些表位于数据口可读区域。

因此 FinSH 应放在最小线程调度成功之后接入。

## 13. RT-Thread 启动流程

不同版本函数名和细节可能略有差异，逻辑顺序通常是：

```text
start.S
  ↓
进入 C 启动入口
  ↓
关闭中断并建立早期栈
  ↓
rt_hw_board_init()
  ├─ 初始化 heap
  ├─ 初始化 UART
  ├─ 初始化 IRQ
  └─ 初始化 Tick timer
  ↓
初始化系统 Tick/软件定时器
  ↓
初始化调度器
  ↓
初始化 idle 线程
  ↓
初始化 timer 线程（若启用）
  ↓
创建 main 线程
  ↓
启动调度器
  ↓
main()
```

中断打开的时机要谨慎：

1. `mtvec` 已设置；
2. 中断栈已设置；
3. handler 已注册；
4. Timer pending 已处理；
5. `mie` 对应位已配置；
6. 最后打开 `mstatus.MIE`。

## 14. `rtconfig.h` 配置什么

`rtconfig.h` 是“这次固件要编进哪些内核功能”的配置结果。第一版可采用最小单核配置。

下面只展示配置方向，具体宏以选定 RT-Thread tag 为准：

```c
#define RT_NAME_MAX                 8
#define RT_ALIGN_SIZE               4
#define RT_THREAD_PRIORITY_MAX      32
#define RT_TICK_PER_SECOND          1000

#define RT_USING_SEMAPHORE
#define RT_USING_MUTEX
#define RT_USING_EVENT
#define RT_USING_MAILBOX
#define RT_USING_MESSAGEQUEUE

#define RT_USING_HEAP
#define RT_USING_SMALL_MEM

#define RT_USING_COMPONENTS_INIT
#define RT_USING_USER_MAIN
#define RT_MAIN_THREAD_STACK_SIZE   2048
#define RT_MAIN_THREAD_PRIORITY     10
```

FinSH 稳定后再加入：

```c
#define RT_USING_FINSH
#define RT_USING_MSH
#define FINSH_USING_MSH
#define FINSH_USING_SYMTAB
#define MSH_USING_BUILT_IN_COMMANDS
```

第一版关闭：

```text
SMP
MMU
FPU 上下文
DFS/文件系统
网络协议栈
动态模块
POSIX 进程
复杂 device class
```

## 15. RT-Thread 不替项目完成什么

RT-Thread 不会自动完成：

- FPGA 上电复位；
- IROM/DRAM BRAM 配置；
- `link.lds` 的地址规划；
- `_start` 入口；
- `.bss` 清零；
- UART 寄存器；
- Timer 比较器；
- 中断控制器；
- CPU 的 `mie/mip/mcause/mepc/mret`；
- ELF 到 COE/MEM 的转换；
- Vivado bitstream 下载。

官方 BSP 只能作为结构参考，因为每块板的存储器和外设不同。

## 16. RT-Thread 与 CoreMark 的关系

CoreMark 可以运行在 RT-Thread 线程中，但二者互不包含：

```text
RT-Thread：提供线程环境、打印和可选命令行
CoreMark：执行 CPU 基准算法并报告结果
```

CoreMark 线程调用 RT-Thread 或 BSP 提供的：

- 打印；
- 计时；
- 静态/动态内存；
- 线程启动。

内核不需要为 CoreMark 修改调度算法。为了便于比较，应记录：

- CoreMark 线程优先级；
- Tick 频率；
- 是否允许其他线程运行；
- 中断是否开启；
- 编译优化；
- CPU 实际频率。

## 17. RT-Thread 层需要完成哪些内容

需要完成：

1. 固定 RT-Thread 版本；
2. 建立最小 `rtconfig.h`；
3. 选中单核 RISC-V common port；
4. 确认无 RV64、RVC、FPU、SMP、RV32A 意外依赖；
5. 与 BSP 对接栈初始化和上下文切换；
6. 接通 `rt_hw_board_init()`；
7. 接通 Timer IRQ 和 `rt_tick_increase()`；
8. 运行两个静态线程；
9. 验证抢占、时间片和 IPC；
10. 最后启用 FinSH。

通常不需要完成：

- 修改通用调度器；
- 编写一个新的操作系统；
- 实现文件系统；
- 实现动态 ELF loader；
- 实现用户进程；
- 实现网络。

## 18. RT-Thread 层的测试顺序

### 18.1 首线程

只运行一个静态线程：

```text
thread A running
```

证明首次上下文恢复成功。

### 18.2 主动切换

线程 A/B 都调用 `rt_thread_yield()`，检查是否交替。

### 18.3 Tick 延时

线程调用 `rt_thread_mdelay()`，检查唤醒时刻。

### 18.4 抢占

高优先级线程被信号量唤醒后，应抢占低优先级线程。

### 18.5 IPC

测试 semaphore、mutex、message queue 的成功、超时和阻塞路径。

### 18.6 长时间运行

至少观察：

- Tick 是否持续增长；
- 栈是否越界；
- ready 队列是否损坏；
- 中断嵌套计数是否回到 0；
- shell 和应用是否都能继续响应。

## 19. 常见故障与所属层

| 现象 | 优先检查 |
|---|---|
| 第一条 C 代码都没执行 | start.S、link.lds、IROM |
| 能打印，调用 delay 后永久停止 | Timer IRQ、mie/mip、Tick ISR |
| 第一次切换就跳飞 | 栈帧、`mepc/mstatus`、context 汇编 |
| 只有一个线程一直运行 | Tick、优先级、ready 队列或未发生阻塞 |
| shell 有输出但键盘无响应 | UART RX、console getchar、RX IRQ |
| 注册的 MSH 命令找不到 | `FSymTab` section 或链接 GC |
| 开启优化后崩溃 | 栈对齐、ABI、未保存寄存器、竞态 |
| 出现非法指令 | `-march` 与 CPU ISA 不一致 |
| CoreMark 能跑，RT-Thread 不稳定 | benchmark 并不能覆盖上下文和 Tick |

## 20. RT-Thread 层完成标准

- 固定版本和配置可以从干净目录重复构建；
- 第一个线程能从构造的初始栈启动；
- 主动切换和中断退出切换都通过；
- Tick、延时和软件定时器可用；
- 信号量、互斥量和至少一种消息 IPC 可用；
- heap 边界正确，或明确采用全静态配置；
- FinSH 可持续接收命令；
- 反汇编中没有 CPU 不支持的指令；
- 不依赖修改内核源码来掩盖 BSP/RTL 问题。

## 21. 官方参考

- [RT-Thread 官方仓库](https://github.com/RT-Thread/rt-thread)
- [RT-Thread Releases](https://github.com/RT-Thread/rt-thread/releases)
- [RT-Thread 内核源码目录](https://github.com/RT-Thread/rt-thread/tree/master/src)
- [RT-Thread RV32 common port](https://github.com/RT-Thread/rt-thread/tree/master/libcpu/risc-v/common)
- [RT-Thread FinSH](https://github.com/RT-Thread/rt-thread/tree/master/components/finsh)

