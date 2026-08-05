# SoC 与 RTOS 移植学习指南

> 适合已有数字 IC、Verilog/SystemVerilog 和计算机组成基础，但对启动代码、链接、异常、中断和 RTOS 不熟悉的读者。  
> 目标不是系统学习一整门操作系统课程，而是尽快获得完成“自研 CPU + SoC + RT-Thread + FPGA 演示”所需的工程认识。

## 1. 这类项目真正跨了哪些知识边界

从 RTL 视角看，CPU 执行 load、store、CSR 和中断响应；从软件视角看，同一组硬件行为会被称作内存、驱动、Trap、系统 Tick 和线程切换。两边描述的是同一条执行链，只是观察位置不同。

```text
C 应用
  ↓ 函数调用、线程 API
RTOS 内核
  ↓ 调度、临界区、上下文切换
BSP / CPU port
  ↓ 启动汇编、Trap、Timer、UART
编译器与链接器
  ↓ ELF、段地址、机器码、内存镜像
SoC
  ↓ 地址译码、中断、存储器、MMIO
CPU 微架构
  ↓ 指令执行、CSR、流水线冲刷、提交
FPGA
```

已经学过计算机组成原理，意味着 PC、寄存器堆、流水线、Cache、异常入口等概念并不陌生。需要补齐的是中间几层的“契约”：

| 契约 | 硬件一侧 | 软件一侧 |
| --- | --- | --- |
| ISA 与特权架构 | 指令、CSR、Trap 状态更新 | 汇编、异常处理、开关中断 |
| ABI | 寄存器和栈都是比特 | 参数传递、调用者/被调用者保存 |
| 内存布局 | ROM、RAM 和 MMIO 地址译码 | 链接脚本、段、堆、栈 |
| 外设接口 | 寄存器与中断线 | 驱动、ISR、console、Tick |
| 启动契约 | reset vector 和存储器初始化 | `start.S`、`.data` 搬运、`.bss` 清零 |
| 调度契约 | 中断和寄存器保存能力 | 线程栈、上下文切换、抢占 |

如果这些契约已经写清，RTOS 不会要求 CPU “识别线程”。CPU 看到的仍是普通指令、内存访问和中断；线程是软件对多份寄存器现场和栈的组织方式。

## 2. 先区分 RTOS 和通用操作系统

### 2.1 当前项目需要的是哪一种能力

典型的 RT-Thread Standard 移植可以运行在：

- 单核 RV32；
- Machine Mode；
- 单一物理地址空间；
- 没有 MMU；
- 没有用户态进程；
- 所有线程共享同一份代码、全局变量和外设地址。

这与 Linux 或完整 xv6 的运行条件相差很大。完成当前项目，不必先实现：

- 页表和虚拟内存；
- 用户态、系统调用和进程隔离；
- ELF 动态加载；
- 文件系统；
- 网络协议栈；
- 多核一致性；
- Supervisor Mode 和 SBI。

这些内容有助于理解通用操作系统，但不是 RTOS 上板的先决条件。

### 2.2 线程和进程不要混在一起

RTOS 线程通常包含：

```text
线程控制块 TCB
├─ 当前栈指针
├─ 栈的起始位置和大小
├─ 优先级
├─ 线程状态
├─ 就绪/等待链表节点
├─ 超时定时器
└─ 入口函数和参数

线程栈
├─ 函数局部变量
├─ 函数返回地址
├─ 编译器保存的寄存器
└─ 人工构造的初始上下文
```

同一 RTOS 中的线程共享：

- 同一地址空间；
- 全局变量；
- 外设寄存器；
- 内核对象；
- 通常也共享代码段。

进程则常带有独立地址空间和权限边界。初次移植 RT-Thread 时，先把“线程 = 一份可暂停、可恢复的寄存器现场 + 独立栈”理解透彻即可。

## 3. 从 C 文件到 CPU 执行的完整过程

### 3.1 编译、汇编、链接和镜像生成

一份固件大致经历：

```text
main.c / board.c / RT-Thread 源码 / start.S
            │
            ▼
        编译与汇编
            │
            ▼
      多个 .o 目标文件
            │
            │  link.lds 决定地址和段布局
            ▼
          firmware.elf
            ├─ 机器码
            ├─ 符号表
            ├─ 调试信息
            ├─ 段地址
            └─ 程序入口
            │
            ├─ objcopy → firmware.bin / firmware.hex / .mem
            ├─ objdump → 反汇编
            └─ size/readelf → 段和符号检查
```

ELF 是最有信息量的产物。`.bin` 和 `.hex` 适合装入存储器，通常丢失符号、段名和调试信息。调试链接和启动问题时，不应只看最终的 `.hex`。

### 3.2 链接脚本做了什么

链接脚本回答两类问题。

第一类是物理存储器：

```ld
MEMORY
{
    IROM (rx)  : ORIGIN = 0x00000000, LENGTH = 256K
    DRAM (rwx) : ORIGIN = 0x80000000, LENGTH = 256K
}
```

第二类是各个程序段放在哪里：

```ld
SECTIONS
{
    .text   : { *(.text.entry) *(.text*) } > IROM
    .rodata : { *(.rodata*) }              > IROM
    .data   : { *(.data*) }                > DRAM AT> IROM
    .bss    : { *(.bss*) *(COMMON) }        > DRAM
}
```

几个容易混淆的概念：

| 名称 | 含义 |
| --- | --- |
| VMA | 程序运行时访问该段的地址 |
| LMA | 镜像中保存该段初始内容的位置 |
| `.text` | 指令 |
| `.rodata` | 只读常量 |
| `.data` | 有非零初值的可写全局/静态变量 |
| `.bss` | 初值为零的全局/静态变量，镜像里通常不保存一大片 0 |
| heap | 动态分配区 |
| stack | 函数调用和线程上下文 |

如果 `.data` 的 LMA 在 ROM、VMA 在 RAM，启动代码必须从 ROM 把初值复制到 RAM。`.bss` 必须由启动代码清零。GNU `ld` 官方文档明确把链接脚本的主要作用描述为控制输入 section 到输出文件和目标内存布局的映射：[GNU ld Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)。

### 3.3 固件怎样进入 FPGA

常见方式有三种：

```text
A. BRAM 预初始化
ELF → HEX/MEM/COE → FPGA BRAM 初始化 → bitstream

B. Bootloader 下载
FPGA 先启动 Boot ROM → 从 UART/SPI 收到固件 → 写入 SRAM → 跳转

C. 外部 Flash
FPGA 配置完成 → CPU 从 Flash XIP，或 Bootloader 搬到 SRAM/DDR
```

早期联调适合 A；频繁改软件时，B 或 C 更省时间。

## 4. 复位后到底发生了什么

### 4.1 硬件阶段

```text
PLL 未锁定或外部复位有效
  → CPU、总线、外设保持复位
  → 时钟稳定
  → 同步释放复位
  → CPU PC = RESET_VECTOR
  → 从 I-ROM 取第一条指令
```

硬件必须保证 reset vector 对应一块可执行存储器。CPU 的参数、SoC 地址译码、ROM 初始化文件和链接脚本入口地址要一致。

### 4.2 软件启动阶段

第一条指令通常位于 `start.S` 或 `crt0.S`：

```text
_start
  ├─ 设置全局指针 gp
  ├─ 设置初始栈指针 sp
  ├─ 可选：关闭中断
  ├─ 搬运 .data
  ├─ 清零 .bss
  ├─ 初始化 Trap 向量
  ├─ 可选：初始化 C 库
  └─ 跳到 C 入口
          ├─ board_init
          ├─ RTOS 内核对象初始化
          ├─ 创建 idle/main/timer 等线程
          └─ 启动第一次调度
```

`main()` 不是处理器规定的入口。CPU 只认 reset vector，`_start` 最终调用 `main()` 是软件工具链和运行库形成的约定。

### 4.3 最值得做的一次练习

写一个只有以下内容的裸机工程：

```text
start.S
link.lds
main.c
uart.c
Makefile
```

要求：

1. `start.S` 设置栈并清零 `.bss`；
2. `main.c` 检查一个零初始化全局变量；
3. 检查一个带初值全局变量；
4. UART 输出二者的值；
5. 用 `readelf -S` 检查段地址；
6. 用 `objdump -d` 找到 `_start` 和 `main`；
7. 用链接 map 文件核对栈顶和各段边界。

完成这个练习后，启动脚本和链接脚本就不再是“编译系统附带的神秘文件”。

## 5. ABI：汇编和 C 能互相调用的基础

ABI 规定函数参数、返回值、栈和寄存器保存规则。RISC-V 整数调用约定中，`a0`–`a7` 用于参数，`a0`、`a1`也用于返回值；`s0`–`s11`属于被调用者保存寄存器，`t0`–`t6`属于临时寄存器。正式定义可查 [RISC-V Procedure Calling Convention](https://docs.riscv.org/reference/abi/v1.0/riscv-cc-procedure-calling-convention.html)。

### 5.1 为什么 RTOS 移植必须理解 ABI

Trap 入口和上下文切换代码要：

- 保存将来还要恢复的寄存器；
- 按 ABI 对齐栈；
- 用 `a0`、`a1` 等向 C 函数传参；
- 调用 C 函数前保护必要现场；
- 恢复 `sp`、`ra`、`mepc` 和状态 CSR；
- 保证新线程第一次恢复时像“刚刚被中断过”一样。

如果遗漏 `s` 寄存器，简单测试可能正常，优化等级提高后会随机损坏局部变量。若栈未按 ABI 对齐，普通整数代码可能能跑，使用库函数或扩展指令时才暴露问题。

### 5.2 调用者保存和被调用者保存

假设函数 A 调用函数 B：

```text
Caller-saved：
  A 如果还想保留 a/t 寄存器，就在 call 前自行保存

Callee-saved：
  B 如果修改 s 寄存器，就必须在返回前恢复
```

线程切换不同于普通函数调用。被切走的线程可能停在任意位置，因此上下文切换通常保存完整的架构现场，而不能只依赖普通函数调用规则。

### 5.3 编译选项必须与 CPU 能力一致

编译器根据 `-march` 决定可以生成哪些指令，根据 `-mabi` 决定寄存器、数据类型和调用约定。32 位无硬件浮点的常见 ABI 是 `ilp32`，但具体 `-march` 必须按 CPU 已实现并验证的扩展填写。

| 扩展 | 对软件的影响 |
| --- | --- |
| `I` | 基础整数指令，RV32 软件的最低基础 |
| `M` | 乘除法；没有时可能调用软件库，也可能因误配置触发非法指令 |
| `A` | 原子指令；单核 RTOS 可用关中断实现部分原子操作，但所选内核代码不能偷偷使用 AMO |
| `C` | 16 位压缩指令；影响取指对齐、`mepc` 步进和反汇编 |
| `F/D` | 浮点寄存器和 ABI；启用后上下文切换还要考虑浮点现场 |
| `Zicsr` | CSR 读写，Trap 和开关中断需要 |
| `Zifencei` | 修改可执行内存后同步取指，Bootloader/代码装载时可能需要 |

常见错误是 CPU 只实现 RV32IM，构建系统却沿用 `rv32imac` 或硬浮点 ABI。程序会在某条压缩、原子或浮点指令处进入 illegal instruction，也可能因为函数参数传递规则不同而在进入 `main()` 前损坏。

移植初期应保存完整编译命令，并用反汇编检查固件中实际出现的指令。RT-Thread、应用、C 库和 CoreMark 必须使用兼容的 ISA 与 ABI 选项。

## 6. Trap、异常和中断

### 6.1 三个词的关系

RISC-V 使用 Trap 统称控制流被硬件转入处理程序的事件：

```text
Trap
├─ Exception：由当前指令同步引起
│  ├─ 非法指令
│  ├─ ECALL
│  ├─ 取指/Load/Store 地址错误
│  └─ 断点
└─ Interrupt：与当前指令异步
   ├─ Timer interrupt
   ├─ Software interrupt
   └─ External interrupt
```

同步异常通常能关联到一条出错指令；中断可能在两条指令之间被接受。

### 6.2 Machine Mode 最少要实现的 CSR 语义

面向 M-mode RTOS，重点包括：

| CSR/机制 | 软件用途 |
| --- | --- |
| `mstatus` | 全局中断使能、Trap 前后状态 |
| `mie` | 各类机器中断使能 |
| `mip` | 中断挂起状态 |
| `mtvec` | Trap 入口 |
| `mepc` | Trap 返回 PC |
| `mcause` | 中断/异常类型 |
| `mtval` | 错误地址或补充信息 |
| `mscratch` | 可用于 Trap 入口交换临时值/栈 |
| `mret` | 恢复特权和中断状态并返回 |

RISC-V 特权规范规定 `mtvec`、Trap 状态更新和 `mret` 行为，当前正式入口见 [Machine-Level ISA](https://docs.riscv.org/reference/isa/priv/machine.html)。

### 6.3 一次中断的硬件—软件分工

硬件负责：

```text
发现 enabled && pending
  → 选择中断
  → 写 mepc
  → 写 mcause
  → 更新 mstatus.MIE/MPIE/MPP
  → PC 跳到 mtvec
```

软件 Trap 入口负责：

```text
保存通用寄存器
  → 读取 mcause
  → 调用 Timer/UART/外部中断处理函数
  → 必要时请求线程切换
  → 恢复寄存器
  → mret
```

中断控制器或外设还要保存 pending，直到软件完成清除。若 Timer 只输出一个很短的脉冲，CPU 关中断期间可能丢失事件。

### 6.4 推荐的裸机中断练习

在引入 RTOS 之前完成：

1. 设置 `mtvec`；
2. 配置 Timer 比较值；
3. 打开对应 `mie` 位；
4. 打开 `mstatus.MIE`；
5. Trap 入口保存寄存器；
6. C 处理函数递增全局计数；
7. 清除或重装 Timer；
8. `mret` 后原程序继续运行；
9. 连续运行数万次中断，检查寄存器和栈不损坏。

如果这一步不稳定，RTOS Tick 和抢占调度也不会稳定。

## 7. RTOS 的五个核心概念

### 7.1 线程栈

每个线程都有独立栈。创建线程时，RTOS 会在栈顶人工放置一份“初始上下文”：

```text
初始 sp
  ↓
┌────────────────────────┐
│ x1/ra                  │
│ x3...x31               │
│ mstatus                │
│ mepc = thread_entry    │
│ a0 = parameter         │
└────────────────────────┘
```

第一次调度到该线程时，汇编恢复这份上下文并执行 `mret` 或等价返回，PC 落到线程入口。对 CPU 来说，这像从一次从未真正发生过的 Trap 中返回。

### 7.2 调度器

调度器维护就绪线程集合，选择最高优先级线程。RT-Thread 常见的是基于优先级的抢占调度：

```text
事件发生
  ├─ 当前线程主动让出
  ├─ 当前线程阻塞
  ├─ 更高优先级线程就绪
  └─ 时间片耗尽
        ↓
比较当前线程和候选线程
        ↓
必要时保存旧 sp、装入新 sp
```

调度策略本身主要是 C 代码；“把 CPU 从线程 A 的寄存器现场切到线程 B”这一小段必须由架构相关汇编完成。

### 7.3 系统 Tick

固定频率 Timer 中断驱动内核时间：

```text
Timer IRQ
  → rt_tick_increase()
  → 延时计数减少
  → 到期线程进入 ready
  → 时间片更新
  → 可能触发调度
```

若 `RT_TICK_PER_SECOND = 1000`，理论上每 1 ms 增加一次 Tick。Timer 的输入时钟和分频值必须与 BSP 宏一致。

### 7.4 临界区

单核无 SMP 系统中，内核常用“暂时关中断”保护短临界区：

```c
level = rt_hw_interrupt_disable();
/* 修改就绪队列或内核对象 */
rt_hw_interrupt_enable(level);
```

这里的返回值不是简单布尔量，而应保存进入临界区之前的中断状态。嵌套关中断时，恢复原状态比无条件开中断安全。

临界区不能长时间执行，不能在里面等待串口发送完，也不能调用会阻塞的接口。

### 7.5 IPC 和阻塞

信号量、互斥量和邮箱的共同机制是：

```text
资源可用
  → 当前线程直接获得

资源不可用
  → 当前线程挂到对象等待队列
  → 状态变为 suspend
  → 调度其他线程
  → 资源到达后重新进入 ready
```

“阻塞”不是 CPU 停止工作，而是当前线程不再占用 CPU。

## 8. RT-Thread 源码应该怎样阅读

官方仓库：[RT-Thread/rt-thread](https://github.com/RT-Thread/rt-thread)。仓库将内核、CPU 架构适配和板级支持分开：

```text
rt-thread/
├─ src/          RT-Thread 内核
├─ include/      内核头文件
├─ libcpu/       CPU 架构相关移植
├─ bsp/          板级支持包
├─ components/   FinSH、文件系统、网络、设备框架等
└─ examples/
```

### 8.1 第一遍只看这些

| 位置 | 要回答的问题 |
| --- | --- |
| `src/thread.c` | 线程怎样创建、启动、挂起和恢复 |
| `src/clock.c` | Tick 怎样推进延时和时间片 |
| `src/timer.c` | 软件定时器如何挂接到 Tick |
| `src/ipc.c` | 线程怎样因 IPC 阻塞并被唤醒 |
| 调度相关源文件 | ready 队列怎样选出下一线程 |
| `include/rtdef.h` | TCB、定时器、IPC 对象有哪些字段 |
| `libcpu/risc-v/common/` | RV32 上下文、Trap 和栈帧怎样实现 |

不要从 `components/` 全部读起。文件系统、网络和 POSIX 会把注意力从最基本的启动、Tick 和线程切换上移开。

### 8.2 RV32 移植最有价值的目录

[RT-Thread `libcpu/risc-v/common`](https://github.com/RT-Thread/rt-thread/tree/master/libcpu/risc-v/common) 当前包含：

| 文件 | 阅读重点 |
| --- | --- |
| `context_gcc.S` | 开关中断和上下文切换 |
| `interrupt_gcc.S` | 中断入口和寄存器保存 |
| `cpuport.c` | 初始线程栈和 CPU 相关接口 |
| `cpuport.h` | XLEN、load/store 和 CSR 辅助 |
| `rt_hw_stack_frame.h` | 汇编栈帧与 C 结构如何一致 |
| `trap_common.c` | RV32 中断注册和分发 |
| `riscv-ops.h` | CSR 操作 |

阅读方法：

1. 先画出 `rt_hw_stack_frame` 的内存布局；
2. 在汇编中标出每个寄存器保存到哪个偏移；
3. 标出旧线程 `sp` 保存到哪里；
4. 标出新线程 `sp` 从哪里取得；
5. 找到第一次启动线程和中断中切换线程的不同入口；
6. 核对 `mstatus`、`mepc` 的保存恢复。

### 8.3 BSP 要实现的最小集合

名称可能随版本和移植方案变化，职责大致固定：

```text
board initialization
├─ 时钟频率常量
├─ UART console
├─ Timer 初始化
├─ 中断入口初始化
└─ heap 边界

CPU port
├─ 初始线程栈
├─ 关中断/恢复中断
├─ 第一次线程切换
├─ 普通线程切换
└─ 中断中的线程切换

drivers
├─ UART putc/getc
├─ Timer ISR
└─ 可选 GPIO 等设备
```

RT-Thread 官方移植文档把 CPU 架构移植放在 `libcpu` 抽象层，并把具体硬件平台放在 BSP 中：[Kernel Porting](https://www.rt-thread.io/document/site/programming-manual/porting/porting/)。

## 9. 六个典型开源项目

### 9.1 PicoRV32 / PicoSoC：最短的软硬件闭环

仓库：[YosysHQ/picorv32](https://github.com/YosysHQ/picorv32)  
重点目录：[picosoc](https://github.com/YosysHQ/picorv32/tree/main/picosoc)

PicoSoC 把一个小型 RISC-V CPU、SPI Flash、SRAM、UART、启动汇编、C 固件和链接脚本放在同一目录，适合第一次看“一个 C 程序怎样进入 SoC”。

建议按这个顺序读：

```text
picosoc/README.md
  → picosoc.v
  → simpleuart.v
  → start.s
  → sections.lds
  → firmware.c
  → Makefile
  → FPGA top / testbench
```

阅读时回答：

- CPU 的 reset vector 在哪里；
- SPI Flash、SRAM、UART 各占什么地址；
- RTL 怎样根据地址选设备；
- `start.s` 怎样进入 C；
- `sections.lds` 怎样匹配硬件地址；
- `.bin/.hex` 怎样生成；
- 仿真怎样把程序送进存储模型。

PicoSoC 很适合学习完整链路，但 PicoRV32 的部分中断机制和接口带有项目自身特点，不能直接当作标准 RISC-V M-mode RTOS 移植模板。

### 9.2 NEORV32：完整度较高的 MCU 型 SoC 参考

仓库：[stnolting/neorv32](https://github.com/stnolting/neorv32)  
文档：[NEORV32 User Guide](https://stnolting.github.io/neorv32/ug/index.html)

NEORV32 是 32 位 MCU 型 RISC-V SoC，包含 CPU、片上存储器、Bootloader、UART、Timer、GPIO、调试和完整软件框架。RTL 使用 VHDL，但系统结构和软硬件接口很值得参考。

关键阅读入口：

```text
rtl/core/              CPU 和 SoC RTL
sw/common/crt0.S       C 运行时入口
sw/common/neorv32.ld   链接脚本
sw/common/common.mk    通用软件构建
sw/lib/                CPU 与外设 HAL/驱动
sw/bootloader/         启动与程序装载
sw/example/            裸机示例
```

官方仓库对 `sw/` 的划分也明确列出了 bootloader、common、image generator 和 peripheral libraries：[NEORV32 software framework](https://github.com/stnolting/neorv32/tree/main/sw)。

从这个项目学习：

- Boot ROM 和应用 IMEM 的区别；
- UART 下载程序与 BRAM 预初始化的区别；
- 外设寄存器如何配套 C 头文件；
- `crt0.S` 与链接脚本怎样约定符号；
- 完整 SoC 怎样维护文档和软件库；
- 仿真、FPGA 和软件构建怎样共用一个工程。

不要逐行模仿 VHDL。先画模块图、地址表和启动流程，再看 wrapper、存储器和外设。

### 9.3 Ibex Simple System / Demo System：SystemVerilog 集成参考

CPU 仓库：[lowRISC/ibex](https://github.com/lowRISC/ibex)  
简单系统：[examples/simple_system](https://github.com/lowRISC/ibex/tree/master/examples/simple_system)  
较完整系统：[lowRISC/ibex-demo-system](https://github.com/lowRISC/ibex-demo-system)

Ibex 是 SystemVerilog 编写的 32 位 RISC-V CPU。`examples/simple_system/rtl/ibex_simple_system.sv` 提供了一个有意保持简单的仿真集成；Demo System 增加外设、调试、FPGA 和软件构建。

适合学习：

- 一个成熟 CPU 核怎样通过 wrapper 接入 SoC；
- 指令口、数据口、握手和错误信号怎样定义；
- CPU 核和 SoC 逻辑为何分目录；
- Verilator 怎样加载 ELF；
- GDB、OpenOCD 和片上调试怎样接入；
- filelist、FuseSoC/CMake 等构建描述怎样管理软硬件。

阅读顺序：

```text
Ibex README / user manual
  → examples/simple_system/rtl/ibex_simple_system.sv
  → Simple System testbench 和 software
  → ibex-demo-system 顶层
  → ibex-demo-system/sw/c
  → Verilator 运行命令
```

Ibex CPU 本体规模较大。当前任务的重点是“如何集成一个 CPU 核”，不必从 IF、ID 到 LSU 全部重读。

### 9.4 RT-Thread 官方仓库：实际移植目标

仓库：[RT-Thread/rt-thread](https://github.com/RT-Thread/rt-thread)  
RV32 通用移植：[libcpu/risc-v/common](https://github.com/RT-Thread/rt-thread/tree/master/libcpu/risc-v/common)  
QEMU RISC-V 参考：[bsp/qemu-virt64-riscv](https://github.com/RT-Thread/rt-thread/tree/master/bsp/qemu-virt64-riscv)

QEMU BSP 的价值是先在已知硬件模型上运行 RT-Thread，观察：

- 构建和配置过程；
- BSP、driver、application 的目录关系；
- `link.lds`、`rtconfig.h` 和构建脚本怎样配合；
- 启动后线程、Tick 和 shell 是什么表现。

该 QEMU BSP 是 RV64，并可能涉及 OpenSBI、S-mode、PLIC 和不同内存布局，不能整目录复制到自研 RV32 M-mode SoC。RV32 架构相关部分以 `libcpu/risc-v/common` 和自身硬件契约为准。

建议先做两件事：

1. 在 QEMU 上运行官方 BSP，创建两个不同优先级线程；
2. 回到源码跟踪 `rt_thread_mdelay()` 如何让线程阻塞、Timer Tick 如何使它重新就绪。

### 9.5 xv6-riscv：补操作系统概念

源码：[mit-pdos/xv6-riscv](https://github.com/mit-pdos/xv6-riscv)  
课程和教材：[MIT 6.1810 xv6](https://pdos.csail.mit.edu/6.1810/2025/xv6.html)

xv6 是教学型 Unix，运行在 RV64 QEMU 上。它不是当前 RTOS 的模板，但几份源码很适合建立操作系统直觉：

| 文件 | 学习内容 |
| --- | --- |
| `kernel/entry.S`、`start.c` | 启动和特权初始化 |
| `kernel/kernelvec.S` | 内核 Trap 保存恢复 |
| `kernel/trap.c` | Trap 原因和设备中断分发 |
| `kernel/swtch.S` | 两个内核上下文怎样切换 |
| `kernel/proc.c` | 进程状态和调度循环 |
| `kernel/uart.c` | UART 驱动 |
| `kernel/plic.c` | 外部中断控制 |
| `kernel/kernel.ld` | 内核链接布局 |

第一次只读 Trap 和 context switch 相关章节。页表、文件系统、系统调用、磁盘和多核锁可以稍后学习。

必须注意：xv6 的 S-mode、用户态、页表和进程模型高于当前 RT-Thread M-mode 单地址空间需求。阅读目的是理解概念，不是复制代码。

### 9.6 LiteX：了解大型 FPGA SoC 生态

仓库：[enjoy-digital/litex](https://github.com/enjoy-digital/litex)

LiteX 用 Python 描述和生成 SoC，支持多种软核、Wishbone/AXI/AHB 等互连、CSR、UART、DRAM 和多种 FPGA 板卡。它适合观察：

- 地址空间和 CSR 如何自动生成；
- 同一外设怎样配套 HDL、C 头文件和文档；
- CPU、总线、DDR、BIOS 和板级工程如何模块化；
- 仿真目标和 FPGA 目标怎样复用 SoC 描述。

当前项目以手写 RTL 为主，因此 LiteX 放在后面看。它展示的是“成熟 SoC 构建系统如何管理复杂度”，不适合拿来替代对地址译码、中断和启动流程的理解。

## 10. 开源项目怎么选

| 目标 | 首选项目 | 原因 |
| --- | --- | --- |
| 看清 C 到 SoC 的最短路径 | PicoSoC | 文件少，RTL、启动、链接和固件在一起 |
| 看完整 MCU 型软硬件组织 | NEORV32 | Bootloader、外设、软件框架和文档完整 |
| 参考成熟 SystemVerilog CPU 接口 | Ibex Simple System | CPU/SoC 边界清楚，验证体系较完整 |
| 完成 RT-Thread 移植 | RT-Thread 官方仓库 | 实际内核和 RV32 通用移植代码 |
| 补 Trap、上下文和调度概念 | xv6-riscv | 教材和代码配套，路径清楚 |
| 了解大规模 FPGA SoC 管理 | LiteX | 自动生成地址、CSR、软件和板级工程 |

最短阅读组合是：

```text
PicoSoC
  → 明白“软件怎样成为 SoC 中的机器码”

RT-Thread libcpu/risc-v/common
  → 明白“线程上下文怎样落到 RISC-V 栈帧”

NEORV32
  → 明白“完整 MCU 型 SoC 怎样组织软硬件”

xv6 的 trap/context switch
  → 补足操作系统直觉
```

Ibex 和 LiteX 用作第二轮工程质量参考。

## 11. 一条两周快速学习路线

这是一条高强度路线，每天约 2～4 小时。时间不足时，以“阶段验收”而不是天数为准。

### 第 1～2 天：工具链和 ELF

学习：

- GCC 的 `-march`、`-mabi`；
- 编译、汇编、链接、objcopy；
- ELF section 和 symbol；
- linker map；
- `.text/.data/.bss/stack`。

练习：

```text
编译一个最小 C 文件
  → 查看反汇编
  → 查看 ELF header
  → 查看 section 地址
  → 查看符号地址
  → 修改 link.lds 后再次比较
```

验收：

- 能解释 ELF 和 HEX 的区别；
- 能从 map 文件找到 `_start`、`main`、栈顶和 `.bss`；
- 能判断链接地址是否落在 SoC 的真实存储器范围。

### 第 3～4 天：裸机启动和 UART

学习：

- reset vector；
- `start.S`；
- ABI；
- MMIO；
- `volatile`；
- UART 轮询输出。

练习：

- 自己写最小 `start.S` 和 `link.lds`；
- UART 输出；
- 验证 `.data` 和 `.bss`；
- 对照 PicoSoC 的 `start.s`、`sections.lds` 和 `firmware.c`。

验收：

- 不借助 RTOS，在 RTL 仿真中稳定打印字符串；
- 复位多次结果一致；
- 能从波形追踪 UART MMIO store。

### 第 5～6 天：Trap 和 Timer

学习：

- `mstatus/mie/mip/mtvec/mepc/mcause/mtval`；
- direct/vectored Trap；
- `mret`；
- 外设 pending；
- ISR。

练习：

- Timer 周期中断；
- ISR 更新计数；
- 主循环继续运行；
- 故意执行非法指令，打印 `mcause/mepc/mtval`。

验收：

- Timer 连续运行不会破坏寄存器；
- 异常和中断能区分；
- `mepc` 返回位置正确；
- 非法地址访问能够进入异常，而不是静默返回。

### 第 7～8 天：线程和上下文切换

学习：

- ABI 栈帧；
- TCB；
- ready/suspend；
- 第一次线程恢复；
- 中断中的抢占。

练习：

- 画出 RT-Thread RV32 stack frame；
- 对照 `context_gcc.S` 和 `interrupt_gcc.S` 标注每条保存恢复；
- 解释旧线程和新线程的 `sp` 放在哪里；
- 在 QEMU RT-Thread 上运行两个线程。

验收：

- 能口头讲清“线程 A 怎样停下，线程 B 怎样从自己的栈恢复”；
- 能解释为什么第一次运行线程也能复用上下文恢复；
- 能定位寄存器保存遗漏会出现什么症状。

### 第 9～10 天：RT-Thread 最小 BSP

学习：

- `libcpu` 与 BSP 的边界；
- `rtconfig.h`；
- board init；
- console；
- Tick；
- heap；
- 构建文件。

练习：

```text
只启用：
  内核
  静态线程
  UART console
  系统 Tick

先关闭：
  文件系统
  网络
  POSIX
  动态模块
  大量设备组件
```

验收：

- idle 和 main 线程能运行；
- 两个线程按预期延时和切换；
- Tick 频率可测；
- 无 shell 时仍能用 UART 日志定位问题。

### 第 11～12 天：SoC 联调

学习：

- 指令/数据 RAM 延迟；
- 地址译码；
- byte enable；
- Timer、UART 中断；
- 仿真内存模型和 FPGA BRAM IP 差异。

练习：

- 运行裸机回归；
- 运行 RTOS 最小镜像；
- 观察从 reset 到第一次线程切换的波形；
- 检查 stack 高水位或填充值；
- 加入 GPIO 线程。

验收：

- 仿真和 FPGA 使用相同地址表；
- 同一 ELF 可以被仿真加载，也能转换成 FPGA 内存镜像；
- RTOS 在板上长时间运行；
- UART 日志无乱码，Tick 周期正确。

### 第 13～14 天：演示和性能

练习：

- 加入 FinSH/MSH；
- 加入 CoreMark；
- 输出频率、周期、迭代次数和编译参数；
- 记录 CPU cycle、instret 和 IPC；
- 准备异常、线程切换和 MMIO 的波形截图。

验收：

- 演示从复位、启动、shell 到多线程和 CoreMark；
- 性能数字可重复；
- 文档中的地址、中断和时钟与 RTL 一致；
- 能解释每一层发生了什么。

## 12. 调试时从哪里切入

### 12.1 完全没有取指

检查：

```text
reset 是否释放
PC 是否等于 reset vector
I-ROM 是否被选中
ROM 初始化文件是否装入
读延迟是否被 CPU 正确等待
第一条指令是否与 ELF 反汇编一致
```

### 12.2 能进入 `_start`，到不了 `main`

检查：

- `sp` 是否在可写 RAM；
- `gp` 是否正确；
- `.data` 源地址和目的地址；
- `.bss` 边界；
- 栈对齐；
- `main` 符号地址和 call 距离；
- CPU 是否支持编译器生成的全部 ISA 扩展。

### 12.3 能跑裸机，RTOS 第一次调度就崩

优先检查：

- 初始线程栈布局；
- `mepc` 是否为线程入口；
- `mstatus` 的返回特权级和中断位；
- `sp` 保存/恢复；
- `ra` 和 `a0`；
- 32 位/64 位寄存器宽度宏；
- 栈是否按 ABI 对齐。

### 12.4 能切一次，Timer 一来就崩

优先检查：

- Trap 入口是否保存全部现场；
- 中断栈和线程栈的使用方式；
- `mcause` 最高位和编号解析；
- Timer pending 是否清除；
- 嵌套中断是否意外开启；
- 中断返回时 `mepc/mstatus` 是否属于当前线程；
- 中断中是否错误地切换了两次。

### 12.5 线程能跑，但延时不准

检查：

- Timer 实际时钟频率；
- compare 增量；
- `RT_TICK_PER_SECOND`；
- RTL 计数器位宽和溢出；
- ISR 是否漏 Tick；
- 仿真时钟单位；
- UART 打印是否严重干扰测量。

## 13. 工程中要形成的五份表

学习最终要落到可共同维护的工程事实：

### 13.1 CPU 能力表

```text
XLEN
支持的 ISA 扩展
特权模式
CSR
异常类型
中断类型
未对齐访问行为
乘除法时序
指令/数据接口
```

### 13.2 内存映射表

```text
Boot ROM
SRAM
UART
Timer
GPIO
Interrupt Controller
Default Error Region
```

### 13.3 中断表

```text
中断号
来源
边沿/电平
使能位置
pending 位置
清除方式
CPU 对应 cause
ISR 名称
```

### 13.4 启动与链接表

```text
reset vector
ELF entry
.text/.rodata
.data VMA/LMA
.bss
heap
main stack
interrupt stack
thread stack
```

### 13.5 时钟复位表

```text
CPU 时钟
UART 时钟
Timer 时钟
PLL lock
复位极性
复位释放顺序
跨时钟域
```

五份表的作用是让 RTL、BSP、链接脚本和测试使用同一组事实。

## 14. 三人团队怎样边学边做

三个人都需要理解完整启动链，但可以各自深入一段。

### 队长：系统接口和集成

主要负责：

- 冻结 CPU 能力、地址、中断、时钟和复位表；
- 保证 RTL、链接脚本和 BSP 一致；
- 维护可运行的主分支和里程碑镜像；
- 跟踪从 reset 到应用的整条路径；
- 组织仿真、上板和最终演示；
- 在两个成员的模块交界处做接口评审。

队长需要能读懂 CPU Trap RTL、上下文汇编、链接脚本和 Timer/UART 驱动，但不必亲自完成所有细节。

### 成员 A：CPU 与 SoC

重点：

- CSR、Trap、`mret`；
- 指令/数据请求握手；
- ROM/RAM；
- MMIO；
- Timer、中断控制；
- Verilator 和 FPGA；
- ISA、裸机和中断回归。

推荐主读 PicoSoC、Ibex Simple System、NEORV32 RTL。

### 成员 B：BSP 与 RT-Thread

重点：

- `start.S`、`link.lds`；
- ABI 和上下文切换；
- RT-Thread `libcpu/risc-v/common`；
- board init；
- UART console；
- Tick；
- 线程、shell 和 CoreMark。

推荐主读 RT-Thread、xv6 的 Trap/context、NEORV32 软件框架。

### 共同检查点

每完成一个里程碑，三人共同回答：

```text
这段代码从哪个地址开始执行？
当前 sp 指向哪里？
若此时发生 Timer IRQ，硬件写哪些 CSR？
Trap 入口保存了哪些寄存器？
当前 MMIO 地址会选中哪个模块？
响应要等待几拍？
出现错误时从日志或波形哪里看？
```

回答不一致时，先修文档和接口，再继续扩展功能。

## 15. 可以暂时不学的内容

为了尽快完成当前项目，可以推迟：

- Linux 内核；
- 完整 xv6 实验；
- 虚拟内存和页表；
- 用户态进程；
- ELF 动态加载；
- SMP；
- Cache coherence；
- 完整 AXI 多主互连；
- 网络和文件系统；
- 复杂低功耗状态；
- 形式化验证的全部方法。

但以下内容不能跳过：

- ABI 和栈；
- ELF、链接脚本和 map；
- 启动代码；
- RISC-V M-mode Trap；
- Timer 和 UART；
- MMIO 和字节写使能；
- 初始线程栈；
- 上下文切换；
- 仿真与 FPGA 存储器时序一致性。

## 16. 最终应达到的认识

完成学习后，应能不看代码画出这条链：

```text
FPGA 复位释放
  → CPU 从 reset vector 取指
  → start.S 建立 C 环境
  → board 初始化 UART 和 Timer
  → RT-Thread 创建线程
  → 调度器装入第一个线程 sp
  → 上下文恢复进入线程入口
  → Timer 产生 Tick
  → Trap 保存当前现场
  → RTOS 更新延时和就绪队列
  → 必要时切换到另一个线程
  → 恢复现场并 mret
  → 应用通过 MMIO 使用 UART/GPIO
```

也应能把一次故障归到某一层：

```text
没有第一条指令       → 复位、地址、ROM、取指接口
进不了 main          → start.S、ABI、链接、RAM
裸机中断失败         → CSR、Timer、Trap、pending
第一次调度失败       → 初始栈和上下文汇编
运行一会儿随机崩溃   → 栈、寄存器保存、越界、竞态
仿真成功而 FPGA 失败 → BRAM/IP 时序、时钟复位、约束
```

达到这个程度，就已经具备完成自研 RV32 SoC 和 RT-Thread 移植的基本认识。余下工作主要是把每一项接口落实、逐层验证，并保留可以重复运行的测试。

## 17. 建议保留的官方资料

- [RISC-V Machine-Level ISA](https://docs.riscv.org/reference/isa/priv/machine.html)
- [RISC-V Procedure Calling Convention](https://docs.riscv.org/reference/abi/v1.0/riscv-cc-procedure-calling-convention.html)
- [GNU ld Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html)
- [QEMU RISC-V System Emulator](https://www.qemu.org/docs/master/system/target-riscv.html)
- [RT-Thread 官方仓库](https://github.com/RT-Thread/rt-thread)
- [RT-Thread Kernel Porting](https://www.rt-thread.io/document/site/programming-manual/porting/porting/)
- [PicoRV32 / PicoSoC](https://github.com/YosysHQ/picorv32)
- [NEORV32](https://github.com/stnolting/neorv32)
- [Ibex](https://github.com/lowRISC/ibex)
- [Ibex Demo System](https://github.com/lowRISC/ibex-demo-system)
- [xv6-riscv 与 MIT 6.1810](https://pdos.csail.mit.edu/6.1810/2025/xv6.html)
- [LiteX](https://github.com/enjoy-digital/litex)
