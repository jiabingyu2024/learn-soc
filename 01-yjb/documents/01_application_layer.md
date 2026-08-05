# 应用层：线程、命令与 CoreMark

> 阅读顺序：本文 → [RT-Thread 层](./02_rt_thread_layer.md) → [BSP 与软件移植层](./03_bsp_porting_layer.md) → [RTL/SoC 层](./04_rtl_soc_layer.md)  
> 总览文档：[SoC 改造、RT-Thread 移植与 CoreMark 接入指南](./rt_thread_soc_porting_guide.md)

## 1. 应用层处在什么位置

应用层是最终实现比赛功能的地方，包括：

```text
applications/
├─ main.c
├─ thread_demo.c
├─ led_cmd.c
├─ ipc_demo.c
└─ coremark_cmd.c
```

应用代码运行在 RT-Thread 创建的线程中，通过 RT-Thread API 使用调度、延时、信号量和消息队列，通过 BSP 或设备驱动访问 UART、LED、按键和定时器。

```text
应用函数
  │
  ├─ 调用 rt_thread_mdelay()、rt_sem_take() 等内核 API
  │
  └─ 调用 led_write()、rt_kprintf() 等板级或设备接口
          ↓
      RT-Thread / BSP
          ↓
      CPU 执行 load/store
          ↓
      SoC MMIO 外设
```

应用层通常不负责：

- 保存和恢复寄存器；
- 设置 `mtvec/mie/mstatus`；
- 产生系统 Tick；
- 直接处理中断入口；
- 规定 `.text/.data/.bss` 的地址；
- 初始化 FPGA BRAM；
- 实现线程调度器。

这些工作分别属于 RT-Thread、BSP 和 RTL/SoC。

## 2. 应用层向下依赖哪些接口

应用层有两类主要接口。

### 2.1 RT-Thread 内核 API

常用接口包括：

| 功能 | 典型接口 | 应用看到的效果 |
|---|---|---|
| 创建线程 | `rt_thread_init/create()` | 建立一个可调度任务 |
| 启动线程 | `rt_thread_startup()` | 线程进入就绪队列 |
| 主动让出 | `rt_thread_yield()` | 允许同优先级线程运行 |
| 延时 | `rt_thread_mdelay()` | 当前线程阻塞一段时间 |
| 信号量 | `rt_sem_take/release()` | 等待或通知一个事件 |
| 互斥量 | `rt_mutex_take/release()` | 保护共享资源 |
| 邮箱 | `rt_mb_send/recv()` | 在线程间传递一个机器字 |
| 消息队列 | `rt_mq_send/recv()` | 在线程间传递结构化消息 |
| 动态内存 | `rt_malloc/free()` | 从 RT-Thread heap 分配内存 |
| 日志输出 | `rt_kprintf()` | 经过 BSP console 输出到 UART |
| 系统 Tick | `rt_tick_get()` | 读取内核时间基准 |

这些 API 是普通 C 函数。编译后，它们和应用一起被链接进同一个 `firmware.elf`。

### 2.2 BSP 或设备接口

应用不应到处直接写：

```c
*(volatile unsigned int *)0x80200040 = value;
```

更适合提供一层板级接口：

```c
void bsp_led_write(rt_uint32_t value);
rt_uint32_t bsp_switch_read(void);
int bsp_uart_putchar(char ch);
```

以后 UART 地址、LED 地址或驱动方式变化，只需修改 BSP，不必修改所有应用。

如果启用 RT-Thread 设备框架，应用也可以使用：

```c
rt_device_find()
rt_device_open()
rt_device_read()
rt_device_write()
rt_device_control()
```

第一版移植可先使用简单 BSP 函数，等内核和中断稳定后再接完整设备框架。

## 3. `main.c` 是如何被执行的

在常见的 RT-Thread 配置中，启动流程会创建一个 main 线程，再在线程环境中调用应用的 `main()`：

```text
CPU 复位
  ↓
start.S
  ↓
RT-Thread 启动
  ↓
初始化调度器、Tick、idle 等
  ↓
创建 main 线程
  ↓
启动调度器
  ↓
main 线程调用 main()
```

因此 `main.c` 不是板子启动后临时下载的程序文件。它在电脑上已经被编译、链接并写进 IROM/DRAM 镜像。

`main()` 的职责通常是：

1. 初始化应用对象；
2. 创建或初始化线程；
3. 启动线程；
4. 返回，或进入一个低频控制循环。

不要把所有功能都写成 `main()` 里的无限循环，否则其他线程很难组织。

## 4. 线程接口是什么样的

RT-Thread 线程入口通常是：

```c
static void thread_entry(void *parameter)
{
    /* 线程代码 */
}
```

接口含义：

```text
输入：parameter，由创建线程的一方提供
返回：无；线程函数通常长期运行
执行环境：拥有自己的栈，由调度器决定何时运行
```

一个线程至少需要以下属性：

| 属性 | 含义 |
|---|---|
| 名称 | 调试和 `ps` 命令中显示 |
| 入口函数 | 第一次获得 CPU 时执行的位置 |
| 参数 | 传给入口函数的 `void *` |
| 栈 | 保存局部变量、调用链和被切换时的寄存器 |
| 优先级 | 数字越小还是越大优先，由 RT-Thread 配置约定；常用配置是数字越小优先级越高 |
| 时间片 | 同优先级线程每次最多连续运行多久 |

### 4.1 静态线程

静态线程的控制块和栈由应用预先分配：

```c
#include <rtthread.h>

static struct rt_thread led_thread;
static rt_uint8_t led_stack[1024];

static void led_thread_entry(void *parameter)
{
    rt_uint32_t value = 1;

    while (1)
    {
        bsp_led_write(value);
        value = (value << 1) ? (value << 1) : 1;
        rt_thread_mdelay(500);
    }
}

static int led_thread_start(void)
{
    rt_err_t err;

    err = rt_thread_init(&led_thread,
                         "led",
                         led_thread_entry,
                         RT_NULL,
                         led_stack,
                         sizeof(led_stack),
                         15,
                         10);
    if (err != RT_EOK)
        return -1;

    return rt_thread_startup(&led_thread);
}
```

优点：

- 内存位置固定；
- 不依赖 heap；
- 早期移植阶段更容易排查。

缺点：

- 栈和控制块长期占用；
- 对象数量需提前确定。

最早的线程切换测试应使用静态线程。

### 4.2 动态线程

动态线程由内核从 heap 分配：

```c
rt_thread_t tid;

tid = rt_thread_create("worker",
                       worker_entry,
                       RT_NULL,
                       2048,
                       12,
                       10);

if (tid != RT_NULL)
    rt_thread_startup(tid);
```

使用动态线程前必须确认：

- 链接脚本给 heap 留出了空间；
- `rt_system_heap_init()` 的起止地址正确；
- `rt_malloc/free()` 经过压力测试；
- 线程退出后的资源回收配置正确。

## 5. 调度、阻塞和抢占在应用中是什么表现

假设有三个线程：

```text
UART RX 线程    优先级 8
控制线程       优先级 12
LED 演示线程   优先级 20
```

UART RX 线程平时等待信号量，不占 CPU：

```text
UART RX 线程等待 semaphore
  ↓
UART 收到字符并产生中断
  ↓
ISR 把字符放入缓冲区并释放 semaphore
  ↓
UART RX 线程成为 ready
  ↓
它的优先级高于当前线程，发生抢占
```

线程“等待”不是不断读取一个变量。正确的阻塞会把线程从 ready 队列移走，CPU 可运行其他线程。

### 5.1 `rt_thread_mdelay()` 发生了什么

应用调用：

```c
rt_thread_mdelay(500);
```

内部过程是：

```text
把 500 ms 换算成 Tick
  ↓
将当前线程挂入定时等待队列
  ↓
当前线程变成 suspend
  ↓
调度另一个 ready 线程
  ↓
Timer IRQ 不断调用 rt_tick_increase()
  ↓
时间到，原线程重新进入 ready 队列
```

如果 Timer IRQ 没有工作，线程会永远睡下去。这种现象通常不是应用 bug，而是 BSP 或 RTL Tick 链路未打通。

## 6. IPC 应该怎么选

### 6.1 信号量

适合表示“某件事发生了”：

```text
UART 收到字符
DMA 完成
按键事件到达
缓冲区有新数据
```

ISR 可以释放信号量，线程等待信号量后处理复杂逻辑。

### 6.2 互斥量

适合保护共享资源：

```text
共享 UART 打印
共享配置结构体
共享 SPI 总线
```

互斥量通常带有优先级继承语义。ISR 中不应获取互斥量。

### 6.3 邮箱

邮箱一般传递一个 `rt_ubase_t` 大小的值，适合：

```text
事件编号
指针
小型状态值
```

### 6.4 消息队列

适合在线程间发送固定大小结构体：

```c
struct app_event
{
    rt_uint32_t type;
    rt_uint32_t value;
};
```

如果应用只有线程演示、FinSH 和 CoreMark，第一版使用信号量和消息队列已经足够。

## 7. FinSH/MSH 命令接在应用层哪里

FinSH/MSH 是 RT-Thread 的可选命令行组件。它运行在一个 shell 线程中，从 console 接收字符，然后查找命令表。

应用可以注册命令：

```c
static int led(int argc, char **argv)
{
    if (argc != 2)
    {
        rt_kprintf("usage: led <hex-value>\n");
        return -1;
    }

    /* 省略字符串转换 */
    bsp_led_write(1);
    return 0;
}

MSH_CMD_EXPORT(led, write LED value);
```

编译时，宏把命令描述放进特殊 section：

```text
FSymTab
```

链接脚本通过：

```ld
KEEP(*(FSymTab))
```

保留命令表。启动后 shell 扫描这张表，找到命令名和函数地址。

因此：

- 命令函数早已存在于固件；
- shell 不会从文件系统装载新的 ELF；
- `FSymTab` 必须位于 CPU 数据口可读的存储器；
- 当前严格哈佛结构下，建议把它初始化到 DRAM。

## 8. CoreMark 在应用层如何接入

CoreMark 是一个应用级基准程序，不属于 RT-Thread 内核。建议目录：

```text
sw/
├─ third_party/
│  └─ coremark/
└─ bsp/superscalar/applications/
   ├─ coremark_port.c
   └─ coremark_cmd.c
```

### 8.1 与普通 `main()` 冲突

CoreMark 自带 `main()`，而应用也有 `main()`。可在编译 CoreMark 时使用：

```text
-Dmain=coremark_main
```

应用包装函数再调用：

```c
extern int coremark_main(void);

static void coremark_entry(void *parameter)
{
    coremark_main();
}
```

### 8.2 作为 MSH 命令运行

```c
static struct rt_thread coremark_thread;
static rt_uint8_t coremark_stack[4096];
static rt_bool_t coremark_running;

static void coremark_entry(void *parameter)
{
    coremark_running = RT_TRUE;
    coremark_main();
    coremark_running = RT_FALSE;
}

static int coremark(void)
{
    if (coremark_running)
    {
        rt_kprintf("CoreMark is already running\n");
        return -1;
    }

    /* 初始化并启动静态线程；实际代码要处理重复运行 */
    return 0;
}

MSH_CMD_EXPORT(coremark, run CoreMark);
```

应区分两套成绩：

| 模式 | 含义 |
|---|---|
| bare-metal CoreMark | 尽量反映 CPU/存储系统本身 |
| RT-Thread CoreMark | 包含 Tick、中断和 RTOS 环境开销 |

应用层只负责启动测试和输出结果。周期计时接口由 CoreMark port/BSP 提供。

## 9. 应用如何访问硬件：一条完整调用链

以输出字符串为例：

```text
应用调用 rt_kprintf("hello")
  ↓
RT-Thread 完成格式化
  ↓
调用 rt_hw_console_output()
  ↓
BSP drv_uart 轮询 TX busy
  ↓
CPU 执行 store 到 UART_TXDATA
  ↓
dmem 请求进入 SocMemBridge
  ↓
地址译码选择 uart_mmio
  ↓
UART 把字节串行发送到 FPGA 引脚
  ↓
电脑串口终端显示 hello
```

以线程延时为例：

```text
应用调用 rt_thread_mdelay()
  ↓
RT-Thread 阻塞当前线程
  ↓
machine_timer 到达 compare
  ↓
timer_irq_i 拉高
  ↓
CPU 保存 mepc/mcause/mstatus 并跳到 Trap
  ↓
BSP Timer ISR 调用 rt_tick_increase()
  ↓
RT-Thread 唤醒到期线程并可能调度
  ↓
汇编恢复目标线程寄存器并执行 mret
```

这两条链路分别覆盖 console 和调度，是应用层最重要的系统验收路径。

## 10. 应用层需要完成哪些内容

第一版建议只做：

```text
main.c
├─ 创建 LED 线程
├─ 创建 thread A / thread B 演示
└─ 打印启动信息

thread_demo.c
├─ 高低优先级演示
├─ mdelay 演示
└─ 信号量演示

led_cmd.c
└─ MSH_CMD_EXPORT(led, ...)

coremark_cmd.c
└─ MSH_CMD_EXPORT(coremark, ...)
```

不要一开始加入网络、文件系统、复杂设备框架和大量动态内存。

## 11. 应用设计时需要明确的资源

### 11.1 线程优先级表

建立一份固定表：

| 线程 | 建议优先级 | 栈 | 说明 |
|---|---:|---:|---|
| UART RX/shell | 8～12 | 2048～4096 B | 输入响应 |
| 控制线程 | 12～16 | 1024～2048 B | 应用主逻辑 |
| CoreMark | 10～14 | 4096 B 起 | 需要实测栈深 |
| LED 演示 | 20～24 | 512～1024 B | 低优先级 |
| idle | 最低 | 由内核配置 | 不做阻塞操作 |

数字仅用于最初规划，最终要通过栈水位和运行行为调整。

### 11.2 栈使用

以下行为会增加线程栈：

- 深层函数调用；
- 大型局部数组；
- 格式化打印；
- CoreMark 局部数据；
- 中断和线程使用同一栈的错误设计。

大型缓冲区应使用静态区或明确的 heap，不要在 1 KiB 线程栈中定义数千字节数组。

### 11.3 共享变量

两个线程共同访问变量时，需要明确：

- 是否只读；
- 是否由单一线程写；
- 是否需要 mutex；
- ISR 是否也会访问；
- 是否应改成消息队列。

`volatile` 只限制编译器优化，不提供线程互斥，也不保证复合操作原子。

## 12. 应用层怎么测试

### 12.1 最小输出

```text
[APP] main entered
[APP] LED thread started
```

证明 main 线程和 console 可用。

### 12.2 时间和调度

两个线程分别打印：

```text
A tick=1000
B tick=1005
A tick=2000
B tick=2010
```

证明 Tick、延时和多个线程基本可用。

### 12.3 抢占

低优先级线程执行循环，高优先级线程等待信号量。释放信号量后，高优先级线程应很快打印响应。

### 12.4 IPC

生产者发送递增序号，消费者检查：

```text
received sequence: 1, 2, 3, ...
```

出现丢号、重复或阻塞时记录线程状态。

### 12.5 CoreMark

验收至少包括：

- CRC/validation 通过；
- 运行超过规定时间；
- 固定编译参数；
- 输出 CPU 频率和模式；
- 连续运行结果差异可解释；
- CoreMark 运行后 shell 仍可继续使用。

## 13. 应用层完成标准

满足以下条件后，应用层可以视为具备比赛展示基础：

- `main()` 在线程环境中正常执行；
- 至少两个线程可延时、切换和抢占；
- 信号量或消息队列演示通过；
- `rt_kprintf()` 稳定输出；
- FinSH 可执行 `help/ps/led/coremark`；
- CoreMark CRC 正确；
- 应用不直接依赖散落的 MMIO 魔法地址；
- 每个线程的优先级、栈和职责有记录；
- 运行一段较长时间后没有死锁、栈溢出或串口失效。
