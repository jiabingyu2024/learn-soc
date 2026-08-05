# BSP 与软件移植层：启动、Trap、驱动和链接

> 上一层：[RT-Thread 层](./02_rt_thread_layer.md)  
> 上上层：[应用层](./01_application_layer.md)  
> 下一层：[RTL/SoC 层](./04_rtl_soc_layer.md)

## 1. BSP 是什么

BSP 是 Board Support Package，中文常译为板级支持包。它把 RT-Thread 的通用要求转换成本项目 CPU 和 SoC 能执行的操作。

```text
RT-Thread 说：“关闭全局中断”
BSP/CPU port 执行：清除 mstatus.MIE

RT-Thread 说：“切换到线程 B”
BSP/CPU port 执行：保存线程 A 的 sp，加载线程 B 的 sp，恢复寄存器

RT-Thread 说：“输出字符”
BSP 驱动执行：轮询 UART STATUS，写 UART TXDATA

RT-Thread 说：“每 1 ms 给我一次 Tick”
BSP 驱动执行：配置 mtimecmp，Timer ISR 调用 rt_tick_increase()
```

BSP 不是 RTL，也不是 RT-Thread 内核。它是二者之间的软件适配层。

## 2. BSP 的上下接口

### 2.1 向上面对 RT-Thread

需要提供：

```text
CPU port
├─ 线程初始栈构造
├─ 主动上下文切换
├─ 中断退出上下文切换
├─ 全局中断开关
└─ Trap 栈帧

board
├─ rt_hw_board_init()
├─ heap 初始化
├─ console 初始化
└─ Tick 初始化

drivers
├─ UART
├─ machine timer
├─ IRQ controller
└─ LED/GPIO 等
```

### 2.2 向下面对 RTL/SoC

BSP 依赖一份固定硬件契约：

- 复位入口地址；
- IROM/DRAM 范围；
- UART、Timer、IRQ 寄存器地址；
- CPU 时钟频率；
- Timer IRQ cause；
- External IRQ cause；
- MMIO 读写时序；
- CSR 支持情况；
- `mtvec` 模式；
- `mret` 行为。

这份契约变化时，BSP、链接脚本、镜像生成和仿真都要重新检查。

## 3. 推荐 BSP 目录

```text
sw/bsp/superscalar/
├─ applications/
│  ├─ main.c
│  ├─ thread_demo.c
│  └─ coremark_cmd.c
│
├─ board/
│  ├─ board.c
│  ├─ board.h
│  ├─ start.S
│  ├─ trap_gcc.S
│  └─ soc.h
│
├─ drivers/
│  ├─ drv_uart.c
│  ├─ drv_timer.c
│  ├─ drv_irq.c
│  └─ drv_gpio.c
│
├─ link.lds
├─ rtconfig.h
├─ rtconfig.py
├─ Kconfig
├─ SConstruct
└─ SConscript
```

第三方源码放在：

```text
sw/third_party/
├─ rt-thread/
└─ coremark/
```

不要直接修改第三方目录来保存本项目地址。自研内容应集中在 BSP 中。

## 4. `soc.h`：软件看到的硬件说明书

建议把地址和寄存器位统一放在：

```c
#ifndef SUPERSCALAR_SOC_H
#define SUPERSCALAR_SOC_H

#include <stdint.h>

#define SOC_CPU_FREQ_HZ       50000000UL

#define SOC_IROM_BASE         0x80000000UL
#define SOC_IROM_SIZE         0x00040000UL
#define SOC_DRAM_BASE         0x80100000UL
#define SOC_DRAM_SIZE         0x00040000UL

#define SOC_LED_BASE          0x80200040UL
#define SOC_UART_BASE         0x80201000UL
#define SOC_TIMER_BASE        0x80202000UL
#define SOC_IRQ_BASE          0x80203000UL

#define MMIO32(addr)          (*(volatile uint32_t *)(uintptr_t)(addr))

#define UART_TXDATA           MMIO32(SOC_UART_BASE + 0x00)
#define UART_RXDATA           MMIO32(SOC_UART_BASE + 0x04)
#define UART_STATUS           MMIO32(SOC_UART_BASE + 0x08)
#define UART_CTRL             MMIO32(SOC_UART_BASE + 0x0c)
#define UART_IRQ_STATUS       MMIO32(SOC_UART_BASE + 0x10)

#define UART_STATUS_TX_BUSY   (1u << 0)
#define UART_STATUS_RX_VALID  (1u << 1)

#define TIMER_MTIME_LO        MMIO32(SOC_TIMER_BASE + 0x00)
#define TIMER_MTIME_HI        MMIO32(SOC_TIMER_BASE + 0x04)
#define TIMER_MTIMECMP_LO     MMIO32(SOC_TIMER_BASE + 0x08)
#define TIMER_MTIMECMP_HI     MMIO32(SOC_TIMER_BASE + 0x0c)
#define TIMER_CTRL            MMIO32(SOC_TIMER_BASE + 0x10)

#endif
```

这段只是接口示例。地址和 bit 定义必须与最终 RTL 一致。

建议同时在 RTL 中建立：

```text
rtl/soc/soc_addr_pkg.sv
```

并由文档或脚本检查 C 与 SystemVerilog 两边的常量。

## 5. `start.S`：复位后的第一段软件

### 5.1 谁调用 `start.S`

FPGA 配置完成、复位释放后，CPU 把 PC 设置为：

```text
0x8000_0000
```

链接脚本通过：

```ld
ENTRY(_start)
```

把 `_start` 设为固件入口，并把 `.start` 放在 IROM 起始位置。

CPU 不是调用 `_start`，而是直接从它的地址取第一条指令。

### 5.2 `start.S` 的职责

按顺序完成：

1. 关闭全局中断；
2. 设置 `gp`；
3. 设置 boot stack；
4. 设置中断栈和 `mscratch`；
5. 处理 `.data` 初值；
6. 清零 `.bss`；
7. 设置 `mtvec`；
8. 清理旧的 pending；
9. 调用 C 启动入口；
10. 若入口意外返回，停在安全循环。

### 5.3 适合当前双镜像方案的骨架

```asm
    .section .start, "ax"
    .align 2
    .globl _start
    .type _start, @function

_start:
    /* mstatus.MIE = 0 */
    csrci mstatus, 0x8
    csrw mie, zero

    /* 设置 gp，禁止 linker relaxation 改写这条序列 */
    .option push
    .option norelax
    la gp, __global_pointer$
    .option pop

    la sp, __boot_stack_top
    andi sp, sp, -16

    la t0, __irq_stack_top
    andi t0, t0, -16
    csrw mscratch, t0

    /* 当前方案中 .data/.rodata 已在 DMEM 镜像中，无需 ROM→RAM copy */

    la t0, __bss_start
    la t1, __bss_end
1:
    bgeu t0, t1, 2f
    sw zero, 0(t0)
    addi t0, t0, 4
    j 1b

2:
    la t0, SW_handler
    csrw mtvec, t0

    call entry

3:
    j 3b
```

如果以后数据口可以读取 IROM，可改成常见方案：

```text
.data 的 LMA 在 IROM
.data 的 VMA 在 DRAM
start.S 从 __data_load_start 复制到 __data_start
```

当前严格哈佛结构不适合直接照搬这段复制逻辑。

## 6. 链接脚本是硬件和软件的地址合同

### 6.1 链接脚本决定什么

`link.lds` 决定：

- `_start` 放在哪里；
- 哪些函数放 IROM；
- 哪些常量和变量放 DRAM；
- `.bss` 的范围；
- RT-Thread 初始化表的范围；
- FinSH 命令表的范围；
- heap 起止地址；
- boot stack 和 IRQ stack；
- 固件是否越界。

### 6.2 当前建议布局

```text
IROM 0x8000_0000..0x8003_FFFF
├─ .start
└─ .text

DRAM 0x8010_0000..0x8013_FFFF
├─ .rt_rodata
│  ├─ .rti_fn*
│  ├─ FSymTab
│  ├─ VSymTab
│  └─ .rodata*
├─ .data/.sdata
├─ .bss/.sbss
├─ heap
├─ boot stack
└─ IRQ stack
```

`.rodata` 放 DRAM 是因为 CPU 读取字符串和常量时走数据口。IROM 目前只有取指端口。

### 6.3 链接脚本骨架

```ld
OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY
{
    IROM (rx)  : ORIGIN = 0x80000000, LENGTH = 256K
    DRAM (rwx) : ORIGIN = 0x80100000, LENGTH = 256K
}

SECTIONS
{
    .start :
    {
        KEEP(*(.start))
        KEEP(*(.text.entry))
    } > IROM

    .text :
    {
        *(.text)
        *(.text.*)
        *(.gnu.linkonce.t.*)
    } > IROM

    .rt_rodata :
    {
        . = ALIGN(8);
        __rt_init_start = .;
        KEEP(*(SORT(.rti_fn*)))
        __rt_init_end = .;

        . = ALIGN(8);
        __fsymtab_start = .;
        KEEP(*(FSymTab))
        __fsymtab_end = .;

        . = ALIGN(8);
        __vsymtab_start = .;
        KEEP(*(VSymTab))
        __vsymtab_end = .;

        *(.rodata)
        *(.rodata.*)
        *(.srodata)
        *(.srodata.*)
    } > DRAM

    .data :
    {
        . = ALIGN(8);
        __data_start = .;
        *(.data)
        *(.data.*)

        PROVIDE(__global_pointer$ = . + 0x800);
        *(.sdata)
        *(.sdata.*)
        __data_end = .;
    } > DRAM

    .bss (NOLOAD) :
    {
        . = ALIGN(8);
        __bss_start = .;
        *(.sbss)
        *(.sbss.*)
        *(.bss)
        *(.bss.*)
        *(COMMON)
        __bss_end = .;
    } > DRAM

    . = ALIGN(8);
    __image_data_end = .;

    __irq_stack_size  = 4K;
    __boot_stack_size = 4K;

    __irq_stack_top    = ORIGIN(DRAM) + LENGTH(DRAM);
    __irq_stack_bottom = __irq_stack_top - __irq_stack_size;
    PROVIDE(__rt_rvstack = __irq_stack_top);

    __boot_stack_top    = __irq_stack_bottom;
    __boot_stack_bottom = __boot_stack_top - __boot_stack_size;

    __heap_start = ALIGN(__image_data_end, 8);
    __heap_end   = __boot_stack_bottom;

    ASSERT(__heap_start < __heap_end,
           "DRAM overflow: no heap/stack space")
}
```

需要根据选定 RT-Thread 版本检查特殊 section 名称，不能只凭这份骨架猜测。

## 7. Trap 和上下文切换

### 7.1 Trap 包含什么

Trap 是异常和中断的统一入口。

```text
同步异常
├─ ECALL
├─ EBREAK
├─ 非法指令
├─ 地址错位
└─ 访问错误

异步中断
├─ Machine software interrupt
├─ Machine timer interrupt
└─ Machine external interrupt
```

CPU 进入 Trap 时，硬件至少应完成：

```text
mepc    ← 返回地址
mcause  ← 异常/中断原因
mtval   ← 补充信息
MPIE    ← MIE
MIE     ← 0
PC      ← mtvec
```

软件 Trap 入口继续保存通用寄存器。

### 7.2 为什么要保存全部寄存器

中断可能在任意普通 C 代码之间发生。编译器不知道中断入口会修改哪些寄存器。因此中断入口一般保存：

```text
x1 ra
x3 gp
x4 tp
x5..x7 t0..t2
x8..x9 s0..s1
x10..x17 a0..a7
x18..x27 s2..s11
x28..x31 t3..t6
mepc
mstatus
```

`x0` 不需要保存，`sp` 由当前栈位置表示。

如果将来增加 FPU，线程使用 FPU 时还要处理浮点寄存器和 `fcsr`。第一版应关闭 FPU 上下文。

### 7.3 中断栈和线程栈

推荐过程：

```text
进入 Trap 时仍在当前线程栈
  ↓
把线程现场压入线程栈
  ↓
csrrw sp, mscratch, sp
  ↓
切到独立 IRQ stack
  ↓
运行 C ISR 和 RT-Thread 中断公共代码
  ↓
切回线程栈
  ↓
恢复寄存器
  ↓
mret
```

这样 ISR 的 C 调用链不会吃掉线程自己的剩余栈空间。

`mscratch` 的具体语义必须与官方 `interrupt_gcc.S` 一致，不能一边把它当 IRQ 栈顶，另一边又把它当普通数据指针。

### 7.4 `trap_gcc.S`

RT-Thread RV32 common port 要求 BSP 提供保存上文后的适配函数。非向量模式可采用：

```asm
    #include "cpuport.h"

    .globl rt_hw_do_after_save_above
    .type rt_hw_do_after_save_above, @function

rt_hw_do_after_save_above:
    addi sp, sp, -4
    STORE ra, 0(sp)

    csrr a0, mcause
    csrr a1, mepc
    mv   a2, sp
    call rt_rv32_system_irq_handler

    LOAD ra, 0(sp)
    addi sp, sp, 4
    ret
```

最终实现应严格匹配所选 RT-Thread tag 的函数签名和栈帧定义。

## 8. `board.c`

`board.c` 负责板级初始化顺序。典型结构：

```c
#include <rtthread.h>
#include "board.h"
#include "soc.h"

extern unsigned char __heap_start[];
extern unsigned char __heap_end[];

void rt_hw_board_init(void)
{
    /* 1. 中断默认关闭 */

    /* 2. 初始化最早可用的 UART TX */
    bsp_uart_init();

    /* 3. 初始化中断注册表/外部中断控制器 */
    bsp_irq_init();

    /* 4. 初始化 heap */
#ifdef RT_USING_HEAP
    rt_system_heap_init(__heap_start, __heap_end);
#endif

    /* 5. 注册并配置系统 Tick timer */
    bsp_tick_init();

    /* 6. 根据版本调用组件初始化所需钩子 */
}
```

不要在 handler 未注册、`mtvec` 未设置时打开全局中断。

`board.c` 还应统一定义：

```c
#define RT_HW_HEAP_BEGIN ((void *)__heap_start)
#define RT_HW_HEAP_END   ((void *)__heap_end)
```

## 9. UART 驱动

### 9.1 第一阶段：轮询发送

```c
void bsp_uart_putchar(char ch)
{
    while (UART_STATUS & UART_STATUS_TX_BUSY)
    {
    }

    UART_TXDATA = (rt_uint32_t)(rt_uint8_t)ch;
}

void rt_hw_console_output(const char *str)
{
    while (*str)
    {
        if (*str == '\n')
            bsp_uart_putchar('\r');

        bsp_uart_putchar(*str++);
    }
}
```

轮询 TX 足以完成：

- 裸机 `hello`；
- RT-Thread 启动日志；
- 初步异常打印。

### 9.2 第二阶段：接收

最简单的 getchar：

```c
char rt_hw_console_getchar(void)
{
    if (UART_STATUS & UART_STATUS_RX_VALID)
        return (char)(UART_RXDATA & 0xff);

    return (char)-1;
}
```

如果 shell 不断轮询，会浪费 CPU。较完整方案是：

1. UART RX 收到字节；
2. 产生 external IRQ；
3. ISR 把字节放入 ring buffer；
4. ISR 释放信号量；
5. shell 或接收线程读取 ring buffer。

ISR 中只做短操作，不要在中断里解析整行命令。

### 9.3 UART 驱动测试

```text
TX 单字节
TX 长字符串
\n 转 \r\n
RX 单字节
RX 连续字符
RX 缓冲区满
RX IRQ pending/clear
同时收发
```

## 10. Timer/Tick 驱动

### 10.1 安全读取 64 位 `mtime`

RV32 上可用：

```c
static rt_uint64_t timer_read_mtime(void)
{
    rt_uint32_t hi0;
    rt_uint32_t lo;
    rt_uint32_t hi1;

    do
    {
        hi0 = TIMER_MTIME_HI;
        lo  = TIMER_MTIME_LO;
        hi1 = TIMER_MTIME_HI;
    } while (hi0 != hi1);

    return ((rt_uint64_t)hi1 << 32) | lo;
}
```

### 10.2 安全写入 `mtimecmp`

为避免写低 32 位时短暂满足比较条件：

```c
static void timer_write_mtimecmp(rt_uint64_t value)
{
    TIMER_MTIMECMP_HI = 0xffffffffu;
    TIMER_MTIMECMP_LO = (rt_uint32_t)value;
    TIMER_MTIMECMP_HI = (rt_uint32_t)(value >> 32);
}
```

### 10.3 初始化 Tick

```c
static rt_uint64_t tick_interval;

void bsp_tick_init(void)
{
    rt_uint64_t now;

    tick_interval = SOC_CPU_FREQ_HZ / RT_TICK_PER_SECOND;
    now = timer_read_mtime();
    timer_write_mtimecmp(now + tick_interval);

    /* 注册 machine timer handler */
    /* 打开 mie.MTIE */
    /* 最后由统一启动流程打开 mstatus.MIE */
}
```

### 10.4 Timer ISR

```c
void bsp_timer_irq_handler(int vector, void *param)
{
    rt_uint64_t next;

    next = timer_read_mtime() + tick_interval;
    timer_write_mtimecmp(next);
    rt_tick_increase();
}
```

为了减少累计漂移，更稳妥的方法是维护上一次 compare：

```text
next_compare += tick_interval
```

如果系统曾长时间关中断，需要处理已经落后多个 Tick 的情况，避免 ISR 立即连续重入。

## 11. IRQ 驱动

### 11.1 第一版

```text
cause 7  → timer handler
cause 11 → UART handler
```

可由统一 Trap 分发：

```c
void machine_irq_dispatch(rt_uint32_t mcause)
{
    rt_uint32_t id = mcause & 0x7fffffffU;

    switch (id)
    {
    case 7:
        bsp_timer_irq_handler(7, RT_NULL);
        break;
    case 11:
        bsp_uart_irq_handler(11, RT_NULL);
        break;
    default:
        /* 记录未处理 IRQ */
        break;
    }
}
```

### 11.2 使用 `trap_common.c`

官方 common port 可以维护 IRQ handler 表：

```c
rt_hw_interrupt_init();
rt_hw_interrupt_install(7, timer_handler, RT_NULL, "timer");
rt_hw_interrupt_install(11, uart_handler, RT_NULL, "uart");
```

是否直接使用这些接口，要以固定版本的 `rthw.h/trap_common.c` 为准。

## 12. 编译配置

### 12.1 ISA 和 ABI

当前 CPU 适合：

```text
-march=rv32im_zicsr
-mabi=ilp32
-mcmodel=medany
```

不要包含：

```text
c  压缩指令
a  原子指令
f/d 浮点指令
```

除非 RTL 已实现并完成验证。

### 12.2 常用 C 参数

```text
-O2
-ffunction-sections
-fdata-sections
-fno-common
-Wall
-Wextra
```

### 12.3 汇编参数

`.S` 文件需要预处理器，因此应由 GCC 编译，而不是直接当普通 `.s`。

### 12.4 链接参数

```text
-nostartfiles
-Wl,-T,link.lds
-Wl,-Map,firmware.map
-Wl,--gc-sections
```

即使使用 `-nostdlib`，通常仍需根据编译结果链接 `libgcc`，否则 64 位除法等辅助函数可能出现未定义引用。更安全的做法是从最小 `-nostartfiles` 开始，明确控制 C 库依赖。

## 13. 从源码到 FPGA 镜像

### 13.1 构建产物

```text
C/汇编源码
  ↓ 编译
*.o
  ↓ link.lds
firmware.elf
  ├─ firmware.map
  ├─ firmware.dis
  ├─ firmware_imem.mem/coe
  └─ firmware_dmem.mem/coe
```

### 13.2 为什么保留 ELF

ELF 包含：

- section 地址；
- 函数符号；
- 入口；
- 调试信息；
- 可加载数据。

MEM/COE 只包含给 BRAM 的数值。调试时应以 ELF 和 MAP 为依据。

### 13.3 `elf2mem.py`

工具按地址分类：

```text
IROM 范围内的可执行 section → IMEM
DRAM 范围内的可加载 section → DMEM
.bss/NOBITS                 → 不生成初值
其他地址                    → 报错
```

必须正确处理：

- RISC-V 小端；
- 字节地址到 32 位 word；
- 非 4 字节整数长度；
- section 间空洞补零；
- IROM/DRAM 越界；
- 两个 section 重叠；
- ELF entry；
- FinSH 和 init table 是否进入 DMEM。

### 13.4 COE 格式

Vivado COE 常见形式：

```text
memory_initialization_radix=16;
memory_initialization_vector=
00000013,
00100093,
...,
0000006f;
```

最后一个值使用分号。Verilator `$readmemh` 更适合使用每行一个 32 位 word 的 `.mem/.hex`。

## 14. 如何送进 FPGA

### 14.1 BRAM 初始化

```text
firmware_imem.coe → IROM Block Memory Generator
firmware_dmem.coe → DRAM Block Memory Generator
                         ↓
                    生成 bitstream
                         ↓
                    JTAG 下载 FPGA
```

bitstream 配置 FPGA 时，BRAM 初值一并生效。

### 14.2 更新程序但不重跑完整综合

后续可使用 Vivado `updatemem` 和 MMI 文件替换 bitstream 中的 BRAM 内容。前提是：

- BRAM 映射稳定；
- MEM 格式正确；
- MMI 与当前实现版本匹配。

第一版先使用完整 bitstream，系统稳定后再优化下载流程。

## 15. 构建脚本分工

### 15.1 `SConstruct/SConscript`

负责：

- RT-Thread 内核源文件；
- RISC-V common port；
- BSP 驱动；
- 应用；
- CoreMark；
- include path；
- 编译参数。

### 15.2 顶层 Makefile

可统一调用：

```text
make sw-build
make sw-disasm
make sw-mem
make sim-rtthread
make fpga-update
```

### 15.3 Vivado Tcl

负责：

- 创建 IROM/DRAM IP；
- 设置宽度、深度、读延迟和 COE；
- 加入新 RTL；
- 设置 FPGA part、PLL、XDC；
- 生成工程和 bitstream。

它不是 GNU linker script。

## 16. BSP 的验证顺序

### 16.1 启动和内存

测试程序检查：

```c
int data_value = 0x12345678;
int bss_value;
const char message[] = "hello";
```

启动后验证：

```text
data_value == 0x12345678
bss_value  == 0
message    可以正确打印
sp         位于 DRAM 栈范围
gp         正确
```

### 16.2 ECALL/MRET

执行 ECALL，Trap 中记录 `mcause/mepc`，调整 `mepc` 后 MRET。

### 16.3 Timer IRQ

裸机环境每 1000 Tick 翻转 LED，验证中断长期稳定。

### 16.4 上下文

为所有寄存器填入特征值，发生一次中断和一次线程切换后检查值是否保持。

### 16.5 UART

测试长字符串、连续输入和 RX IRQ。

### 16.6 RT-Thread

顺序验证：

```text
首线程
主动 yield
Tick delay
抢占
信号量
FinSH
CoreMark
```

## 17. 常见问题

| 现象 | 可能原因 |
|---|---|
| PC 从 0x80000000 开始但马上非法指令 | IMEM 字节序、`-march`、COE 索引 |
| 字符串读取为 0 | `.rodata` 被放进数据口不可读的 IROM |
| `.data` 初值错误 | DMEM 镜像未包含 `.data` |
| `.bss` 随机 | start.S 未清零或边界符号错误 |
| 第一次 C 调用崩溃 | sp/gp/ABI/对齐错误 |
| 第一次中断崩溃 | mtvec、mscratch、栈帧或 CSR 错误 |
| MRET 后跳错位置 | mepc 保存语义不对 |
| delay 永远不返回 | timer IRQ 或 `rt_tick_increase()` 未工作 |
| MSH 命令消失 | `--gc-sections` 删除命令表，缺少 KEEP |
| FPGA 与 Verilator 表现不同 | BRAM 读延迟/READ_FIRST/初始化文件不一致 |
| 优化等级改变后崩溃 | 汇编未遵守 ABI、未保存寄存器或竞态 |

## 18. BSP 层需要完成什么

必须完成：

- `soc.h`；
- `link.lds`；
- `start.S`；
- 与 common port 匹配的 `trap_gcc.S`；
- `board.c/h`；
- UART TX/RX 驱动；
- machine timer/Tick 驱动；
- IRQ 分发；
- RT-Thread 配置和构建脚本；
- ELF 到 IMEM/DMEM 的转换；
- 仿真和 FPGA 共用的镜像规则。

可以后做：

- 完整设备框架；
- UART DMA；
- 文件系统；
- Bootloader；
- 动态模块；
- 网络；
- 多核。

## 19. BSP 完成标准

- 从复位到 C 入口可重复；
- `.data/.bss/.rodata` 正确；
- boot stack、IRQ stack、heap 不重叠；
- ECALL、Timer IRQ、External IRQ、MRET 正确；
- 主动和中断上下文切换都通过；
- UART console 双向可用；
- `link.lds` 溢出会在链接阶段报错；
- ELF、MAP、反汇编和双存储器镜像都可自动生成；
- 同一固件在 Verilator 和 FPGA 上得到一致启动日志。

## 20. 参考

- [RT-Thread RV32 common port](https://github.com/RT-Thread/rt-thread/tree/master/libcpu/risc-v/common)
- [RT-Thread RISC-V BSP 链接脚本示例](https://github.com/RT-Thread/rt-thread/blob/master/bsp/qemu-virt64-riscv/link.lds)
- [RT-Thread FinSH](https://github.com/RT-Thread/rt-thread/tree/master/components/finsh)

官方 QEMU BSP 可能是 RV64、统一内存、MMU 或不同中断控制器，只能参考结构，不能直接复制地址和宏。

