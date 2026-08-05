# CoreMark 使用、源码结构与 SoC/RT-Thread 接入指南

> 用户所说的 “call mark” 在本文中按 **CoreMark** 理解。  
> 官方仓库：[eembc/coremark](https://github.com/eembc/coremark)  
> 官方介绍：[EEMBC CoreMark](https://www.eembc.org/coremark/)  
> 本地源码副本：[references/coremark-official](./references/coremark-official/)  
> 本文核对的官方提交：`1f483d5`，检出日期：2026-07-28。

本文主要回答三个问题：

1. CoreMark 到底怎样使用，最后会得到什么；
2. 官方仓库里每个文件负责什么，程序是怎样运行起来的；
3. 怎样把它接入自研 RV32 SoC，以及怎样作为 RT-Thread 中的一条命令运行。

---

## 1. 先给出结论

CoreMark 是一个用 ANSI C 编写的嵌入式 CPU 基准程序。它不是操作系统，也不是 CPU 验证集，更不是需要额外连接到 RTL 总线上的硬件模块。

接入后的实际关系是：

```text
CoreMark C 源码
    ↓ 和 RT-Thread、BSP 一起编译
coremark.o + RT-Thread 内核对象 + BSP 对象
    ↓ 由 link.lds 链接
一个完整的 ELF 固件
    ↓ objcopy / bin2hex / 生成 COE
IROM 初始化文件或可下载镜像
    ↓
FPGA 上电，CPU 从复位地址取指
    ↓
启动 RT-Thread
    ↓
在 MSH 中输入 coremark ...
    ↓
RT-Thread 中的 CPU 线程执行 CoreMark
    ↓
UART 打印 CRC、运行时间和 Iterations/Sec
```

因此，CoreMark 的“接入”主要是一个软件集成问题：

- 保留官方算法源码；
- 自己编写平台适配文件 `core_portme.c` 和 `core_portme.h`；
- 把计时接口连接到 `mtime`、`mcycle` 或 RT-Thread Tick；
- 把输出接口连接到 UART/Console；
- 处理 CoreMark 自带 `main()` 与 RT-Thread 应用入口的重名；
- 把全部源码加入构建系统，并确认 ROM、RAM 和线程栈足够。

对当前比赛项目，推荐采用以下顺序：

```text
PC 上原版 CoreMark
    → 自研交叉工具链编译
    → SoC 裸机短迭代运行
    → RTL 仿真中检查 CRC
    → FPGA 裸机运行至少 10 秒
    → 接入 RT-Thread 命令
    → FPGA 上进行正式性能测试
```

不要一上来就把 CoreMark、RT-Thread、FinSH 和新 SoC 同时接入。否则出现异常时，很难判断问题在 CPU、链接脚本、计时器、操作系统还是 CoreMark port。

---

## 2. CoreMark 是什么

### 2.1 它主要测什么

CoreMark 的目标是用一个很小、可移植、可自校验的程序，观察嵌入式处理器的基本整数执行性能。它会涉及：

- 整数加减、乘法和移位；
- 条件分支和循环；
- 函数调用；
- 指针访问；
- Load/Store；
- 链表访问；
- 矩阵运算；
- 状态机；
- CRC 计算。

对自研 RV32 CPU 来说，它会综合反映：

- 流水线是否能持续取指和执行；
- 分支跳转代价；
- 前递、暂停和相关处理；
- 乘法器实现；
- IROM、DRAM 和缓存的访问延迟；
- 编译器生成代码的质量；
- 中断和 RTOS 调度对程序的干扰。

最终最常看的数字是：

```text
Iterations/Sec
```

即每秒完成多少次 CoreMark 迭代。数值越大，表示在该测试条件下完成相同工作越快。

还经常看到：

```text
CoreMark/MHz = Iterations/Sec ÷ CPU 频率（MHz）
```

它把分数除以 CPU 主频，用来粗略比较每 MHz 的执行效率。例如：

```text
CPU 频率 = 50 MHz
Iterations/Sec = 75

CoreMark/MHz = 75 ÷ 50 = 1.5
```

### 2.2 它不能证明什么

CoreMark 跑通不等于 CPU 已经完全正确。

它不能替代：

- RISC-V ISA 测试；
- CSR、异常和中断专项测试；
- 随机指令验证；
- 内存一致性验证；
- RT-Thread 上下文切换验证；
- 外设和 DMA 验证；
- 长时间稳定性测试。

CoreMark 的程序规模较小，官方也明确指出，它主要面向处理器核心、整数操作、基本访存和控制流。对带 Cache 的高性能 CPU，代码和数据可能全部进入一级 Cache，因此它不能充分代表大容量 DDR、复杂 Cache 层次或真实大型应用。

更准确的理解是：

```text
ISA 测试回答：CPU 是否按指令集规则执行？
RTOS 测试回答：中断、调度和上下文切换是否正确？
CoreMark 回答：在一组固定的嵌入式整数负载下，整个执行路径有多快？
```

### 2.3 为什么它还会检查 CRC

如果基准程序只做运算、不使用结果，编译器可能把部分运算优化掉；如果 CPU 执行错误，也可能仍然打印出一个看似正常的时间。

CoreMark 会把各算法的结果送入 CRC，并与标准值比较。只有：

- 输入种子正确；
- 数据规模正确；
- CPU 执行正确；
- 数据类型宽度正确；
- 算法源码没有被错误修改；
- CRC 匹配；
- 正式测试时间不少于 10 秒；

才会输出：

```text
Correct operation validated.
```

所以查看 CoreMark 结果时，不能只抄 `Iterations/Sec`，还必须看验证是否成功。

---

## 3. CoreMark 内部到底包含哪些工作负载

官方介绍通常把它概括为四类算法：

```text
链表处理
矩阵运算
状态机
CRC
```

从源码的算法掩码看，主要算法编号实际是三个：

```c
#define ID_LIST   (1 << 0)
#define ID_MATRIX (1 << 1)
#define ID_STATE  (1 << 2)
```

CRC 既是负载的一部分，也是贯穿各算法的结果校验手段。

### 3.1 链表：`core_list_join.c`

链表部分会进行：

- 链表初始化；
- 根据索引或数据查找节点；
- 遍历链表；
- 反转链表；
- 使用归并排序重新排列链表；
- 对部分结果进行 CRC。

它主要给处理器带来：

- 指针追踪；
- 非连续内存访问；
- 大量 Load；
- 数据相关分支；
- 函数调用；
- 排序带来的控制流变化。

这部分对 Load 延迟、前递、分支预测和数据 RAM 访问较敏感。

### 3.2 矩阵：`core_matrix.c`

矩阵部分包含：

- 矩阵加常数；
- 矩阵乘常数；
- 向量与矩阵运算；
- 矩阵乘法；
- 结果累计和 CRC。

它主要覆盖：

- 紧密循环；
- 数组顺序访问；
- 整数乘法；
- 累加；
- 循环分支。

如果 CPU 的 M 扩展乘法指令没有实现，或者编译参数没有声明 M 扩展，编译器可能调用软件乘法库，分数会明显下降。

### 3.3 状态机：`core_state.c`

状态机部分会解析一组类似数字的字符串，并判断它们属于：

- 整数；
- 小数；
- 科学计数法；
- 非法格式。

程序会修改部分输入、再次解析，最后恢复数据并计算 CRC。

它主要覆盖：

- `switch`；
- 多层 `if/else`；
- 数据相关分支；
- 字节 Load/Store；
- 状态转移。

这部分对控制冒险和分支处理较敏感。

### 3.4 CRC：`core_util.c`

CRC 相关函数包括：

- `crcu8()`；
- `crc16()`；
- `crcu16()`；
- `crcu32()`。

CRC 的作用有两层：

1. 它本身是一种常见嵌入式整数负载；
2. 它把运行结果压缩为可检查的值，防止错误执行被当成有效成绩。

### 3.5 一次迭代不是四个相互独立的程序

在 `core_main.c` 中，`iterate()` 每轮会两次调用链表 benchmark。链表 benchmark 在工作过程中又会根据配置调用矩阵和状态机部分，所有结果最终参与 CRC。

可以把它理解为：

```text
iterate()
└─ 重复 iterations 次
   ├─ core_bench_list(..., 1)
   │  ├─ 链表查找、排序等
   │  ├─ core_bench_matrix(...)
   │  └─ core_bench_state(...)
   ├─ CRC 累计
   ├─ core_bench_list(..., -1)
   │  └─ 再执行一组相关操作
   └─ CRC 累计
```

---

## 4. 官方仓库的内容结构

本地副本位于：

```text
learn/references/coremark-official/
```

当前官方仓库根目录大致如下：

```text
coremark/
├─ core_main.c
├─ core_list_join.c
├─ core_matrix.c
├─ core_state.c
├─ core_util.c
├─ coremark.h
├─ coremark.md5
├─ Makefile
├─ README.md
├─ barebones_porting.md
├─ LICENSE.md
│
├─ barebones/
│  ├─ core_portme.c
│  ├─ core_portme.h
│  ├─ core_portme.mak
│  └─ ee_printf.c
│
├─ simple/
├─ linux/
├─ posix/
├─ cygwin/
├─ freebsd/
├─ macos/
├─ rtems/
├─ zephyr/
└─ docs/
```

### 4.1 必须编译的通用源码

| 文件 | 作用 | 移植时是否应修改 |
| --- | --- | --- |
| `core_main.c` | 主流程、迭代次数、计时、CRC 检查和结果打印 | 不应修改 |
| `core_list_join.c` | 链表负载 | 不应修改 |
| `core_matrix.c` | 矩阵负载 | 不应修改 |
| `core_state.c` | 状态机负载 | 不应修改 |
| `core_util.c` | 种子解析、CRC、类型检查等 | 不应修改 |
| `coremark.h` | 公共类型、结构体和函数声明 | 不应修改 |

官方 `coremark.md5` 保存了上述六个文件的摘要。运行官方 Makefile 的 `make check` 会检查它们是否发生变化。

正式成绩要求不能修改这些通用算法文件。平台差异应全部放在 port 层、构建参数和链接脚本中。

### 4.2 平台适配目录

每个平台目录一般提供：

```text
core_portme.h
core_portme.c
core_portme.mak
```

三者分别负责：

| 文件 | 职责 |
| --- | --- |
| `core_portme.h` | 数据类型、浮点能力、内存方式、种子方式、多线程方式等编译配置 |
| `core_portme.c` | 计时、初始化、收尾、内存分配和并行执行接口 |
| `core_portme.mak` | 编译器、编译参数、链接参数、运行命令 |

`barebones/` 是自研 SoC 最合适的起点，但它只是模板。模板故意保留了 `#error`，要求开发者真正实现计时和 UART，而不是直接编译。

### 4.3 `README.md` 与 `barebones_porting.md`

- `README.md`：运行命令、参数、有效成绩规则、报告格式；
- `barebones_porting.md`：无操作系统或小型嵌入式系统的 port 方法；
- `docs/`：其他说明资料；
- `linux/`、`posix/`：学习成熟平台怎样实现计时和多上下文；
- `rtems/`、`zephyr/`：观察 RTOS 集成思路。

移植时不要把所有平台目录都加入工程。一个目标平台只能选择或编写一套 `core_portme.*`。

---

## 5. CoreMark 的完整执行流程

CoreMark 自带的 `main()` 位于 `core_main.c`。其主要流程是：

```text
main()
├─ portable_init()
├─ 获取 seed1、seed2、seed3、iterations
├─ 获取一块 2000 Byte 的数据内存
├─ 将内存划分给链表、矩阵和状态机
├─ 初始化各算法的数据
├─ 如果 iterations=0，则自动估算迭代次数
├─ start_time()
├─ iterate()
├─ stop_time()
├─ get_time()
├─ time_in_secs()
├─ 根据标准 seed 检查各算法 CRC
├─ 打印 Iterations/Sec、编译参数、内存位置和 CRC
├─ portable_fini()
└─ 返回
```

### 5.1 自动迭代次数

如果第四个参数 `iterations` 为 0，CoreMark 会先试跑：

```text
1 次 → 10 次 → 100 次 → 1000 次 → ...
```

直到试跑时间至少达到约 1 秒，再估算正式运行所需次数，使正式运行通常落在 10 秒量级。

这要求计时器在一次进程中可以反复执行：

```text
start_time()
stop_time()
start_time()
stop_time()
...
```

如果硬件计时器不能被反复读取，或者 RTL 仿真太慢，应显式指定 `ITERATIONS`。

### 5.2 标准数据规模

标准运行要求：

```c
TOTAL_DATA_SIZE = 2000
```

这 2000 Byte 是全部算法共享的数据区，不是每个算法各 2000 Byte。程序会根据启用的算法数量进行划分。默认三个主要算法都启用，所以输出中经常看到：

```text
CoreMark Size : 666
```

这表示每个主要算法分到约 666 Byte，不代表总数据区只有 666 Byte。

---

## 6. Port 层需要提供哪些接口

CoreMark 与 SoC、BSP 或 RT-Thread 的边界集中在 port 层。

### 6.1 计时接口

必须实现：

```c
void start_time(void);
void stop_time(void);
CORE_TICKS get_time(void);
secs_ret time_in_secs(CORE_TICKS ticks);
```

调用关系是：

```text
start_time()
    保存起始计数值

stop_time()
    保存结束计数值

get_time()
    返回结束值 - 起始值

time_in_secs()
    根据计数器频率把 ticks 换算成秒
```

最关键的约束是：

```text
EE_TICKS_PER_SEC 必须等于计时源每秒实际增加的次数
```

例如：

| 计时源 | 每秒计数 | `EE_TICKS_PER_SEC` |
| --- | ---: | ---: |
| 50 MHz 的 `mcycle` | 50,000,000 | 50,000,000 |
| 1 MHz 的 `mtime` | 1,000,000 | 1,000,000 |
| 1 kHz RT-Thread Tick | 1,000 | 1,000 |
| 100 Hz 定时器 | 100 | 100 |

如果实际计数器是 50 MHz，却错误配置成 1 MHz，打印时间和分数会相差 50 倍。

### 6.2 平台初始化和结束接口

必须实现：

```c
void portable_init(core_portable *p, int *argc, char *argv[]);
void portable_fini(core_portable *p);
```

裸机版本的 `portable_init()` 可能需要初始化：

- 时钟；
- UART；
- Cache；
- 硬件计时器；
- 性能计数器。

RT-Thread 版本调用 CoreMark 时，板级时钟、UART 和系统 Tick 通常已经初始化。因此这里不应再次复位 UART 或重配系统定时器，只需检查类型、准备 CoreMark 私有状态。

`portable_init()` 最后应设置：

```c
p->portable_id = 1;
```

`portable_fini()` 则恢复为 0。

### 6.3 输出接口

CoreMark 通过：

```c
int ee_printf(const char *fmt, ...);
```

输出结果。

裸机上可采用：

- 官方 `barebones/ee_printf.c`；
- 自己的 UART printf；
- 半主机或仿真器字符输出。

RT-Thread 上可采用：

- 工具链 libc 的 `printf`；
- `rt_kprintf`；
- 官方 `ee_printf.c`，并把 `uart_send_char()` 接到 RT-Thread Console。

需要特别检查 `%f`：

- `HAS_FLOAT=1` 时，CoreMark 会用浮点格式打印时间和分数；
- 某些精简 `rt_kprintf` 不支持 `%f`；
- 若输出出现 `%f` 原样字符或错误数字，应改用支持浮点的 `ee_printf`；
- `HAS_FLOAT=0` 可以先跑通，但时间和分数按整数秒计算，精度较低。

浮点计算发生在计时区之外。RV32 没有 F 扩展并不妨碍算法运行，只是格式化和分数换算可能由软件库完成。

### 6.4 内存接口

CoreMark 支持三种数据区来源：

```c
MEM_STATIC
MEM_MALLOC
MEM_STACK
```

| 方式 | 特点 | 对当前项目的建议 |
| --- | --- | --- |
| `MEM_STATIC` | 使用静态全局数组，最容易控制地址 | 首次 RT-Thread 接入推荐 |
| `MEM_MALLOC` | 运行时申请，需实现 `portable_malloc/free` | heap 稳定后使用 |
| `MEM_STACK` | 2000 Byte 数据区放在线程栈 | 容易造成 FinSH 线程栈溢出 |

RT-Thread 中如果采用 `MEM_MALLOC`，可连接：

```c
#define portable_malloc rt_malloc
#define portable_free   rt_free
```

但必须确认：

- RT-Thread heap 已初始化；
- `TOTAL_DATA_SIZE=2000`；
- 没有并发重复运行；
- 输出中准确说明数据位于哪类 RAM。

### 6.5 种子接口

CoreMark 支持：

```c
SEED_ARG       /* 从 argc/argv 获取 */
SEED_FUNC      /* 从平台函数获取 */
SEED_VOLATILE  /* 从 volatile 变量获取 */
```

裸机没有命令行时，通常使用 `SEED_VOLATILE`。

RT-Thread 启用 FinSH/MSH 后，推荐使用 `SEED_ARG`。因为 MSH 命令本身就提供 `argc/argv`，可以在同一固件中分别运行性能参数和验证参数。

---

## 7. 先在电脑上怎样使用

这一步的目标不是测试自研 CPU，而是先认识正常输出。

在 Linux、WSL 或带有 GNU Make/GCC 的环境中进入官方仓库：

```bash
cd coremark
make
```

默认会生成：

```text
run1.log    性能参数运行
run2.log    验证参数运行
```

常用目标包括：

```bash
make run
make run1.log
make run2.log
make check
make clean
```

如果没有 Make，也可以直接编译：

```bash
gcc -O2 -o coremark \
  core_list_join.c \
  core_main.c \
  core_matrix.c \
  core_state.c \
  core_util.c \
  simple/core_portme.c \
  -DPERFORMANCE_RUN=1 \
  -DITERATIONS=1000
```

这一步应重点观察：

- 输出包含哪些字段；
- 性能运行和验证运行的 seed 有何不同；
- `Correct operation validated` 在什么情况下出现；
- 修改 `ITERATIONS` 后运行时间怎样变化。

PC 结果只用于熟悉 CoreMark，不能代表 RV32 SoC 的成绩。

---

## 8. 怎样接入裸机 RV32 SoC

即使最终目标是 RT-Thread，也强烈建议先做裸机接入。

### 8.1 推荐目录

```text
software/
├─ common/
│  ├─ start.S
│  ├─ trap.S
│  ├─ link.lds
│  └─ crt0.c
│
├─ bsp/
│  ├─ uart.c
│  ├─ uart.h
│  ├─ timer.c
│  └─ timer.h
│
├─ third_party/
│  └─ coremark/
│     ├─ core_main.c
│     ├─ core_list_join.c
│     ├─ core_matrix.c
│     ├─ core_state.c
│     ├─ core_util.c
│     └─ coremark.h
│
├─ coremark_port/
│  ├─ core_portme.c
│  ├─ core_portme.h
│  └─ ee_printf.c
│
└─ Makefile
```

建议把官方算法源码和自己写的 port 分开。以后更新官方源码时，不会覆盖平台代码。

### 8.2 裸机 `core_portme.h` 的主要配置

第一版可采用：

```c
#define HAS_FLOAT       1
#define HAS_TIME_H      0
#define USE_CLOCK       0
#define HAS_STDIO       0
#define HAS_PRINTF      0

#define SEED_METHOD     SEED_VOLATILE
#define MEM_METHOD      MEM_STATIC
#define MULTITHREAD     1

#define MAIN_HAS_NOARGC 1
#define TOTAL_DATA_SIZE 2000

#define COMPILER_VERSION "riscv-none-elf-gcc <实际版本>"
#define COMPILER_FLAGS   "-O2 -march=rv32im_zicsr -mabi=ilp32"
#define MEM_LOCATION     "Code in IROM, data in on-chip DRAM"
```

`-march` 必须只声明 CPU 真正实现的扩展。例如没有压缩指令就不能写 `rv32imc`，没有 F 扩展就不能写 `rv32imaf`。

### 8.3 裸机计时源的选择

优先级建议：

```text
稳定的 64 位 mtime
    > 可正确读取的 mcycle/mcycleh
    > 独立的固定频率硬件计数器
    > 中断累计的软件 Tick
```

若使用 RV32 上的 64 位计数器，不能简单先读低 32 位再读高 32 位，因为低位可能在两次读取之间溢出。应采用：

```c
static unsigned long long read_counter64(void)
{
    unsigned int hi1, lo, hi2;

    do
    {
        hi1 = read_counter_hi();
        lo  = read_counter_lo();
        hi2 = read_counter_hi();
    } while (hi1 != hi2);

    return ((unsigned long long)hi2 << 32) | lo;
}
```

如果使用 `mcycle` CSR，则 `read_counter_hi/lo()` 对应 `mcycleh/mcycle`；如果使用 MMIO `mtime`，则对应 `MTIME_HI/MTIME_LO`。

### 8.4 裸机镜像怎样运行

裸机 CoreMark 与普通 C 程序一样：

```text
start.S
    设置 sp、清零 .bss、复制 .data
    ↓
调用 CoreMark main()
    ↓
算法运行
    ↓
通过 UART 输出
```

编译和加载过程为：

```text
riscv GCC 编译 .c/.S
    ↓
link.lds 生成 coremark.elf
    ↓
objcopy 生成 coremark.bin
    ↓
bin2hex 或工具脚本生成 IROM hex/coe
    ↓
Verilator/Vivado 加载到 IROM
    ↓
CPU 从复位向量运行
```

CoreMark 不需要自己实现新的链接脚本。它使用该 SoC 裸机程序已有的 `link.lds`。但链接脚本必须有足够的 IROM、DRAM、栈和堆空间。

---

## 9. 怎样接入 RT-Thread

### 9.1 它在 RT-Thread 中属于哪一层

CoreMark 属于应用层：

```text
应用层
└─ CoreMark
   ├─ 官方算法源码
   ├─ MSH 命令包装
   └─ RT-Thread CoreMark port

RT-Thread 层
├─ 线程与调度
├─ Console/FinSH
├─ Tick
└─ heap（可选）

BSP 层
├─ UART
├─ Timer
├─ 中断
└─ board.c

RTL/SoC 层
├─ CPU
├─ IROM/DRAM
├─ mcycle 或 mtime
└─ MMIO UART
```

CoreMark 不随 RT-Thread 内核每个 Tick 自动运行。它只是被编译进 RT-Thread 固件中的一个普通 C 程序，只有创建线程或执行命令时才运行。

### 9.2 推荐目录

```text
rt-thread/
└─ bsp/
   └─ superscalar/
      ├─ applications/
      │  ├─ main.c
      │  └─ coremark_cmd.c
      │
      ├─ coremark/
      │  ├─ upstream/
      │  │  ├─ core_main.c
      │  │  ├─ core_list_join.c
      │  │  ├─ core_matrix.c
      │  │  ├─ core_state.c
      │  │  ├─ core_util.c
      │  │  └─ coremark.h
      │  │
      │  ├─ port/
      │  │  ├─ core_portme.c
      │  │  ├─ core_portme.h
      │  │  └─ ee_printf.c
      │  │
      │  └─ SConscript
      │
      ├─ drivers/
      ├─ link.lds
      ├─ rtconfig.h
      └─ SConstruct
```

其中：

- `upstream/` 保存官方原文件；
- `port/` 保存本项目自行实现的适配；
- `coremark_cmd.c` 只负责把 CoreMark 暴露成 MSH 命令；
- `SConscript` 负责把源码加入 RT-Thread 构建。

### 9.3 处理两个 `main()` 的冲突

RT-Thread BSP 已经有应用入口：

```c
int main(void);
```

CoreMark 的 `core_main.c` 也定义了：

```c
int main(...);
```

两者直接一起链接会出现：

```text
multiple definition of `main`
```

不建议直接修改官方 `core_main.c`。推荐只对 CoreMark 源码组增加编译宏：

```text
-Dmain=coremark_main
```

预处理后，CoreMark 的入口变为：

```c
int coremark_main(int argc, char *argv[]);
```

算法源码本身没有被手工改写，后续仍可通过官方 MD5 检查。

如果采用无参数裸机模式：

```text
-DMAIN_HAS_NOARGC=1
```

入口则是：

```c
int coremark_main(void);
```

### 9.4 推荐使用 MSH 参数作为 seed

RT-Thread 中建议配置：

```c
#define SEED_METHOD     SEED_ARG
#define MAIN_HAS_NOARGC 0
```

命令包装可以写成：

```c
#include <rtthread.h>
#include <finsh.h>

extern int coremark_main(int argc, char *argv[]);

static int coremark_cmd(int argc, char **argv)
{
    return coremark_main(argc, argv);
}

MSH_CMD_EXPORT_ALIAS(coremark_cmd, coremark, run CoreMark benchmark);
```

这样可以在同一个固件中运行：

```text
msh /> coremark 0 0 0x66 0
msh /> coremark 0x3415 0x3415 0x66 0
```

四个参数依次为：

```text
seed1 seed2 seed3 iterations
```

第四个参数为 0 时自动估算迭代次数。

如果 RTL 仿真只想跑 10 次：

```text
msh /> coremark 0 0 0x66 10
```

这可以用于功能验证，但运行不足 10 秒，不能报告为正式 CoreMark 成绩。

### 9.5 不要忽略 FinSH 线程栈

若上面的命令直接调用 `coremark_main()`，程序运行在 FinSH 的 `tshell` 线程中。

风险包括：

- `MEM_STACK` 会额外占用约 2000 Byte；
- printf 格式化还需要栈；
- CoreMark 内部有较多局部变量和函数调用；
- FinSH 默认线程栈不一定足够；
- CoreMark 运行期间 Shell 会阻塞，不能继续输入命令。

第一版可采用：

```text
MEM_STATIC
适当增大 FINSH_THREAD_STACK_SIZE
只允许一次 CoreMark 运行
```

更规范的实现是让命令创建专用线程：

```text
coremark 命令
    ↓
检查是否已有 benchmark 在运行
    ↓
复制参数
    ↓
创建 coremark_worker 线程
    ↓
在线程中调用 coremark_main()
```

专用线程栈建议先给 8～16 KiB，再通过 map、栈水位或溢出检查逐步缩小。具体值应以实际工具链和 printf 实现为准，不能只照抄。

### 9.6 `SConscript` 的组织方式

以下是结构示意，具体 API 以当前 BSP 的 RT-Thread 构建脚本为准：

```python
from building import *

cwd = GetCurrentDir()

coremark_src = [
    cwd + '/upstream/core_main.c',
    cwd + '/upstream/core_list_join.c',
    cwd + '/upstream/core_matrix.c',
    cwd + '/upstream/core_state.c',
    cwd + '/upstream/core_util.c',
    cwd + '/port/core_portme.c',
    cwd + '/port/ee_printf.c',
]

coremark_group = DefineGroup(
    'CoreMarkCore',
    coremark_src,
    depend=['BSP_USING_COREMARK'],
    CPPPATH=[
        cwd + '/upstream',
        cwd + '/port',
    ],
    CPPDEFINES=[
        'main=coremark_main',
        'TOTAL_DATA_SIZE=2000',
        'MULTITHREAD=1',
    ],
    CCFLAGS=' -O2'
)

Return('coremark_group')
```

`coremark_cmd.c` 应放在另一个源码组中，不能也带上 `main=coremark_main` 宏。

正式测试时应确认：

- 五个通用 `.c`；
- `coremark.h`；
- `core_portme.c`；
- 所使用的 `ee_printf`；

都采用一致且已记录的关键优化参数。官方运行规则要求所有基准源码使用相同编译参数。

### 9.7 RT-Thread port 的推荐配置

第一版建议：

```c
#define HAS_TIME_H      0
#define USE_CLOCK       0
#define HAS_STDIO       0
#define HAS_PRINTF      0

#define SEED_METHOD     SEED_ARG
#define MEM_METHOD      MEM_STATIC
#define MULTITHREAD     1
#define MAIN_HAS_NOARGC 0
#define TOTAL_DATA_SIZE 2000

#define COMPILER_VERSION "riscv-none-elf-gcc <实际版本>"
#define COMPILER_FLAGS   "-O2 -march=rv32im_zicsr -mabi=ilp32"
#define MEM_LOCATION     "Static data in on-chip DRAM"
```

当 heap、线程栈和输出都稳定后，再考虑 `MEM_MALLOC` 或其他方案。

### 9.8 RT-Thread 下选择哪种计时器

#### 方案 A：读取硬件 `mtime`

推荐用于正式成绩。

优点：

- 频率明确；
- 通常为 64 位；
- 不依赖线程是否被调度；
- 可以把中断和 RTOS 干扰自然计入总耗时；
- 与 CPU 主频解耦时也容易解释。

要求：

- `mtime` 在运行过程中连续递增；
- 明确其频率，不要默认它一定等于 CPU 频率；
- RV32 读取 64 位值时使用高—低—高重读；
- `time_in_secs()` 使用真实频率换算。

#### 方案 B：读取 `mcycle/mcycleh`

也适合正式成绩。

优点：

- 能直接得到处理器经历的周期数；
- 便于计算 CoreMark/MHz；
- 不需要额外 MMIO。

要求：

- CPU 正确实现 `mcycle/mcycleh`；
- 计数器不会在异常或中断后错误清零；
- 读取 64 位计数值的方法正确；
- CPU 主频在测试中固定；
- 明确计数器在停顿、WFI 和调试状态下的行为。

#### 方案 C：`rt_tick_get()`

适合首轮接入和功能验证：

```c
start = rt_tick_get();
...
elapsed = rt_tick_get() - start;
seconds = elapsed / RT_TICK_PER_SECOND;
```

它的分辨率取决于系统 Tick。例如 1000 Hz 时分辨率是 1 ms。运行 10 秒以上时误差可以较小，但它会把 Tick 中断和其他线程的影响一起计入。

正式比赛结果最好使用硬件计时器，并用示波器、板上 GPIO 翻转或外部秒表校验一次“计时 10 秒是否真的约等于 10 秒”。

---

## 10. CoreMark 对 SoC 硬件提出了什么要求

CoreMark 本身不要求新增专用外设，但为了稳定运行，SoC 至少应具备：

### 10.1 CPU 指令能力

最低要覆盖交叉编译器实际生成的指令，包括：

- RV32I 基本整数指令；
- Load/Store；
- 分支和跳转；
- 函数调用所需的 ABI 行为；
- 若 `-march` 带 `m`，则实现乘除法；
- 若启用 `zicsr`，则实现所用 CSR 指令。

CoreMark 不直接要求操作系统中断，但 RT-Thread 运行需要正确的 CSR、Trap 和定时器中断。

### 10.2 IROM

IROM 需要放入：

- RT-Thread 内核代码；
- BSP；
- FinSH；
- CoreMark 算法；
- printf 或 `ee_printf`；
- 编译器运行库。

接入后应检查：

```bash
riscv-none-elf-size rtthread.elf
```

以及链接 map 中的：

```text
.text
.rodata
```

若 IROM 只有 16 KiB，RT-Thread + FinSH + CoreMark 通常非常紧张。应根据实际 ELF 结果扩展 IROM，不能只估算源文件大小。

### 10.3 DRAM

DRAM 需要同时容纳：

- `.data`；
- `.bss`；
- RT-Thread heap；
- 主栈；
- idle 线程栈；
- FinSH 线程栈；
- CoreMark 专用线程栈；
- CoreMark 2000 Byte 数据区；
- 设备对象和内核对象。

要重点检查：

```text
__bss_end
heap 起点/终点
各线程栈
系统栈
CoreMark 静态数据
```

它们不能重叠。

### 10.4 计时器

至少提供一种可读、频率稳定、能覆盖 10 秒以上的计数器：

- `mtime`；
- `mcycle`；
- 自定义 64 位自由运行计数器；
- RT-Thread Tick。

假设使用 32 位、50 MHz 的自由运行计数器：

```text
溢出时间 = 2^32 ÷ 50,000,000
         ≈ 85.9 秒
```

它能覆盖一次约 10 秒的测试，但自动迭代或低性能仿真可能接近溢出边界。64 位计数器更稳妥。

### 10.5 UART

UART 只负责输出最终结果，并不位于主要计时区间内，因此波特率通常不会直接决定 CoreMark 分数。

但 UART 仍必须：

- 能稳定打印长行；
- 不丢字符；
- 换行正确；
- 不在 `portable_init()` 中意外复位 RT-Thread Console；
- 在仿真和 FPGA 上采用一致或明确区分的输出路径。

---

## 11. 链接脚本需要怎样处理

CoreMark 不提供一份通用于所有 SoC 的 `link.lds`。它使用 BSP 或裸机工程已有的链接脚本。

最基础的链接关系是：

```text
CoreMark .text/.rodata → IROM
CoreMark .data/.bss    → DRAM
CoreMark 静态数据区    → .bss
CoreMark 线程栈        → RT-Thread 管理的 RAM
```

通常不需要为 CoreMark新建链接脚本，只需要确认现有脚本：

```ld
MEMORY
{
    IROM (rx)  : ORIGIN = ..., LENGTH = ...
    DRAM (rwx) : ORIGIN = ..., LENGTH = ...
}
```

具有足够空间，并正确放置：

```ld
.text
.rodata
.data
.bss
```

如果希望专门研究 IROM/DRAM 放置对性能的影响，可以在 map 和链接脚本中按对象文件建立 section。例如：

```ld
.coremark_text :
{
    *core_main.o(.text .text.*)
    *core_list_join.o(.text .text.*)
    *core_matrix.o(.text .text.*)
    *core_state.o(.text .text.*)
    *core_util.o(.text .text.*)
} > IROM
```

但正式报告必须说明代码和数据实际位于：

- 片上 BRAM；
- 片外 DDR；
- Cache；
- TCM；
- Flash；
- 其他存储器。

不能只写“RAM”而不说明访问条件。

---

## 12. RTL 仿真应该怎样跑

CoreMark 的完整正式运行要求至少 10 秒。对 RTL 仿真而言，这可能意味着极长的主机运行时间。

因此要把仿真分为两类。

### 12.1 功能仿真

目的：

- 确认程序能启动；
- 确认 IROM/DRAM 地址正确；
- 确认 Load/Store、分支、乘法工作；
- 确认 UART 输出；
- 确认 CRC 匹配。

使用较小迭代次数：

```text
ITERATIONS=1
ITERATIONS=10
```

或者 MSH 中：

```text
coremark 0 0 0x66 10
```

这种结果一定会提示不足 10 秒，不能作为分数，但非常适合 RTL 回归。

### 12.2 性能仿真

目的：

- 统计 CoreMark 每次迭代的仿真周期；
- 观察流水线停顿、分支失败、访存等待；
- 对比 RTL 优化前后的周期数。

可以不等待墙钟 10 秒，而是记录：

```text
固定 iterations
总 CPU cycles
cycles / iteration
```

这不是官方 CoreMark 成绩，却很适合 CPU 微结构优化：

```text
cycles_per_iteration = total_cycles / iterations
```

每次仿真必须保持：

- 相同程序镜像；
- 相同编译参数；
- 相同迭代次数；
- 相同存储器延迟；
- 相同 Cache 初始状态；
- 相同中断配置。

### 12.3 正式分数放到 FPGA 实机

最终正式测试应在 FPGA 上运行：

- 至少 10 秒；
- 两组标准 seed 都通过；
- 使用稳定的硬件计时器；
- 记录 CPU 实际频率；
- 记录编译器和参数；
- 记录代码、数据和 Cache 放置；
- 重复运行多次。

---

## 13. 怎样判断结果有效

### 13.1 两组必须通过的标准参数

官方运行规则要求总数据区为 2000 Byte，并验证两组 seed：

性能参数：

```text
seed1 = 0
seed2 = 0
seed3 = 0x66
TOTAL_DATA_SIZE = 2000
```

验证参数：

```text
seed1 = 0x3415
seed2 = 0x3415
seed3 = 0x66
TOTAL_DATA_SIZE = 2000
```

RT-Thread MSH 中可分别运行：

```text
coremark 0 0 0x66 0
coremark 0x3415 0x3415 0x66 0
```

### 13.2 2K 标准运行的关键 CRC

性能参数的典型检查值：

```text
seedcrc   = 0xe9f5
crclist   = 0xe714
crcmatrix = 0x1fd7
crcstate  = 0x8e3a
```

验证参数的典型检查值：

```text
seedcrc   = 0x18f2
crclist   = 0xe3c1
crcmatrix = 0x0747
crcstate  = 0x8d84
```

`crcfinal` 与迭代次数有关，不应只照抄另一个平台的值。

### 13.3 正式结果的必要条件

至少满足：

- 正式运行不少于 10 秒；
- 性能和验证两组 seed 均通过；
- `TOTAL_DATA_SIZE=2000`；
- `ee_u8/ee_s16/ee_u16/ee_s32/ee_u32` 位宽正确；
- 官方六个通用源码未修改；
- 所有基准源码使用一致编译参数；
- 计时器换算正确；
- 输出 `Correct operation validated`。

### 13.4 运行时间不足 10 秒

如果输出：

```text
ERROR! Must execute for at least 10 secs for a valid result!
```

说明程序未必有功能错误，只是该次结果不能正式报告。处理方法：

- 第四个参数使用 0，让程序自动估算；
- 或逐步增大固定 `ITERATIONS`；
- 检查 `EE_TICKS_PER_SEC` 是否写错；
- 检查计时器是否在运行。

### 13.5 正确的报告内容

不要只报告：

```text
CoreMark = 100
```

至少记录：

```text
CPU/SoC：
CPU 频率：
ISA 与微结构版本：
FPGA 型号：
CoreMark 官方提交：
编译器版本：
完整编译参数：
Iterations：
Total time：
Iterations/Sec：
CoreMark/MHz：
seedcrc：
crclist：
crcmatrix：
crcstate：
代码存储位置：
数据存储位置：
Cache 状态：
RT-Thread 是否运行：
系统 Tick 频率：
测试时是否有其他线程或中断：
```

官方推荐的基本格式是：

```text
CoreMark 1.0 : N / Compiler and flags / Memory placement
```

报告 CoreMark/MHz 时，还要说明存储器频率与核心频率的比例，以及可配置 Cache 的频率关系。

---

## 14. RT-Thread 下怎样定义公平的测试环境

同一个 SoC 可以有两种都合理、但含义不同的测试方式。

### 14.1 模式 A：尽量测 CPU 核心

配置建议：

- 单独的高优先级 CoreMark 线程；
- 运行时不启动其他业务线程；
- 禁止在计时区内打印调试日志；
- CPU 主频固定；
- Cache 状态固定；
- Tick 和必要中断保持可解释；
- 重复运行 3～5 次。

它更接近“CPU + 编译器 + 基础存储系统”的性能。

### 14.2 模式 B：测运行 RTOS 时的系统表现

配置建议：

- RT-Thread Tick 正常运行；
- Console 和必要设备正常开启；
- 保留指定的后台线程；
- 记录 CoreMark 线程优先级；
- 记录其他线程负载。

这个分数包含 RTOS、中断和调度带来的开销。

两种模式不能混在一张表中直接比较。报告中应明确写：

```text
Core-only-like mode
```

或：

```text
RT-Thread active mode
```

不建议为了得到更高分，在未说明的情况下关闭全部中断 10 秒。那既破坏 RTOS 的正常语义，也可能让结果失去可比性。

---

## 15. 常见问题与定位方法

### 15.1 链接时报两个 `main`

现象：

```text
multiple definition of `main`
```

原因：RT-Thread 应用入口和 CoreMark 入口重名。

处理：

```text
只给 CoreMark 源码组增加 -Dmain=coremark_main
```

### 15.2 找不到 `core_portme.h`

原因：

- port 目录没有加入头文件路径；
- 同时加入了多个平台 port；
- `coremark.h` 先包含了错误版本。

处理：

```text
CPPPATH 中只保留当前平台的 port/
```

### 15.3 编译遇到 `#error`

原因：直接使用了 `barebones/core_portme.c` 模板，其中的计时器和 UART 故意没有实现。

处理：复制到自己的 `port/`，实现计时、初始化和输出。

### 15.4 程序卡死

按以下顺序判断：

1. UART 是否在进入 CoreMark 前还能打印；
2. 固定 `ITERATIONS=1` 是否结束；
3. 是否卡在自动估算迭代次数；
4. `time_in_secs()` 是否永远返回 0；
5. 计时器是否真的递增；
6. CPU 是否陷入非法指令 Trap；
7. 乘法或除法软件库是否链接；
8. 线程栈是否溢出；
9. DRAM 地址或字节使能是否错误。

### 15.5 CRC 不匹配

优先检查：

- `ee_s16/ee_u16/ee_s32/ee_u32` 位宽；
- `ee_ptr_int` 是否能保存指针；
- 字节和半字 Load/Store；
- 符号扩展；
- 非对齐访问处理；
- 大小端；
- 乘法结果；
- 编译器声明的 ISA 是否超过 RTL；
- RAM 写掩码；
- 数据区是否与栈或 heap 重叠。

可以使用：

```text
-DCORE_DEBUG=1
```

将迭代次数缩短到 1，配合指令 trace 和波形定位。

### 15.6 分数异常高

可能原因：

- `EE_TICKS_PER_SEC` 配大；
- 计时器走得比声明的慢；
- 计时器在部分时间停止；
- 运行时间不足 10 秒；
- 修改了算法源码；
- 数据规模不是 2000；
- 编译器把 seed 当作编译期常量；
- 输出的 CPU 频率不是实际频率。

### 15.7 分数异常低

可能原因：

- 使用 `-O0`；
- M 扩展没有启用，乘法变成软件调用；
- IROM/DRAM 每次访问等待很多周期；
- 分支处理代价高；
- 流水线频繁停顿；
- CoreMark 在线程中被其他任务抢占；
- UART 调试输出被错误放进计时区；
- Cache 没开或反复失效；
- CPU 实际频率低于标称频率。

### 15.8 没有浮点输出

现象：

```text
Total time (secs): %f
```

或结果乱码。

原因：`HAS_FLOAT=1`，但 `ee_printf` 不支持 `%f`。

处理：

- 使用官方 `barebones/ee_printf.c` 并实现字符输出；
- 或链接支持浮点格式化的 libc；
- 或第一版设置 `HAS_FLOAT=0`，先验证功能。

### 15.9 Shell 运行后系统崩溃

优先怀疑：

- FinSH 线程栈太小；
- `MEM_STACK` 占用过大；
- 递归或 printf 使用较多栈；
- CoreMark 被重复并发启动；
- heap 还未初始化；
- port 中重新初始化了 RT-Thread 正在使用的 UART/Timer。

---

## 16. 对当前项目最合适的落地步骤

### 阶段 1：认识官方程序

- 使用本地 `references/coremark-official`；
- 在 PC 上执行官方 `make`；
- 阅读 `run1.log` 和 `run2.log`；
- 确认 `make check` 通过。

### 阶段 2：裸机短测试

- 复制六个官方通用文件；
- 基于 `barebones/` 编写自己的 port；
- 使用 `MEM_STATIC`；
- 使用 UART 输出；
- 先设 `ITERATIONS=1`；
- 在 RTL 仿真中检查标准 CRC。

### 阶段 3：裸机 FPGA 正式测试

- 校准硬件计时器；
- 自动选择或手动调整迭代次数；
- 运行至少 10 秒；
- 性能、验证两组 seed 都通过；
- 记录编译器、内存和频率。

### 阶段 4：接入 RT-Thread

- 把官方源码放入 `upstream/`；
- 把 RT-Thread port 放入 `port/`；
- 使用 `-Dmain=coremark_main`；
- 添加 `coremark_cmd.c`；
- 首先使用 `MEM_STATIC`；
- 使用 MSH 参数传入 seed；
- 检查 FinSH/CoreMark 线程栈。

### 阶段 5：性能分析

在同样的软件镜像和测试参数下，记录：

- `mcycle`；
- `minstret`；
- IPC；
- 分支数和错误预测数；
- Load/Store 等待周期；
- IROM/DRAM stall；
- 总 Iterations/Sec。

这样可以把“分数下降”进一步拆成具体 RTL 原因，而不是只看一个最终数字。

---

## 17. 哪些文件复制，哪些文件自己写

| 类别 | 文件 | 处理方式 |
| --- | --- | --- |
| 官方算法 | `core_main.c` | 原样保留 |
| 官方算法 | `core_list_join.c` | 原样保留 |
| 官方算法 | `core_matrix.c` | 原样保留 |
| 官方算法 | `core_state.c` | 原样保留 |
| 官方算法 | `core_util.c` | 原样保留 |
| 官方公共头 | `coremark.h` | 原样保留 |
| 平台适配 | `core_portme.h` | 参考 barebones，自行配置 |
| 平台适配 | `core_portme.c` | 自行实现 Timer、初始化、内存等 |
| 输出适配 | `ee_printf.c` | 使用官方版本或自行连接 Console |
| 构建 | `SConscript`/Makefile | 自己写 |
| RT-Thread 命令 | `coremark_cmd.c` | 自己写 |
| 链接 | `link.lds` | 使用并检查 BSP 现有脚本 |
| 镜像转换 | bin/hex/coe 脚本 | 使用项目现有流程 |

一句话总结：

```text
算法不改，port 自己写，入口做包装，和 RT-Thread 一起链接。
```

---

## 18. 最终验收清单

### 源码

- [ ] 官方六个通用文件未修改；
- [ ] 记录 CoreMark commit；
- [ ] 只有一套 `core_portme.*` 被编译；
- [ ] 可以执行官方 MD5 检查。

### 编译

- [ ] `-march` 与 CPU 实现一致；
- [ ] `-mabi` 正确；
- [ ] CoreMark 各源码使用一致优化参数；
- [ ] map 中没有未解析的软件乘除法函数；
- [ ] `main` 已安全改名为 `coremark_main`。

### 存储

- [ ] `TOTAL_DATA_SIZE=2000`；
- [ ] IROM 容量足够；
- [ ] DRAM 容量足够；
- [ ] CoreMark 数据区不与栈、heap 重叠；
- [ ] 线程栈经过检查。

### 计时

- [ ] 计时源持续递增；
- [ ] 计时频率配置正确；
- [ ] RV32 读取 64 位计数器的方法正确；
- [ ] 外部校验 10 秒时间基本准确；
- [ ] 正式运行时间不少于 10 秒。

### 验证

- [ ] 性能 seed 通过；
- [ ] 验证 seed 通过；
- [ ] 输出 `Correct operation validated`；
- [ ] CRC 与标准值匹配；
- [ ] RTL 短迭代回归稳定；
- [ ] FPGA 重复运行结果稳定。

### 报告

- [ ] 记录 CPU 和 FPGA 版本；
- [ ] 记录实际频率；
- [ ] 记录工具链和完整参数；
- [ ] 记录 IROM/DRAM/Cache 条件；
- [ ] 记录 RT-Thread、Tick 和后台线程状态；
- [ ] 同时给出 Iterations/Sec 与 CoreMark/MHz；
- [ ] 没有把不足 10 秒的仿真结果当正式成绩。

---

## 19. 建议优先阅读的官方文件

按学习顺序：

1. [官方 README](./references/coremark-official/README.md)：先看运行规则和结果格式；
2. [裸机移植指南](./references/coremark-official/barebones_porting.md)：理解 port 边界；
3. [core_main.c](./references/coremark-official/core_main.c)：理解完整执行流程；
4. [coremark.h](./references/coremark-official/coremark.h)：理解配置和数据结构；
5. [barebones/core_portme.h](./references/coremark-official/barebones/core_portme.h)：整理自己的平台宏；
6. [barebones/core_portme.c](./references/coremark-official/barebones/core_portme.c)：实现 Timer 和初始化；
7. 三个算法源码：最后再研究链表、矩阵和状态机细节。

第一次学习不需要从 `core_list_join.c` 第一行开始逐句读。先抓住：

```text
main 流程
→ port 接口
→ 计时
→ seed
→ 内存
→ CRC
→ 正式运行规则
```

就已经具备把 CoreMark 接到自研 SoC 和 RT-Thread 中的基本认识。

