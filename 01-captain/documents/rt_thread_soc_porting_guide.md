# superScalar SoC 改造、RT-Thread 移植与 CoreMark 接入指南

> 适用工程：`superScalar`，当前分支 `dev-v4.0`  
> 当前目标：在现有 RV32 处理器和 FPGA SoC 上完成裸机 C、机器模式中断、RT-Thread、串口命令行和 CoreMark  
> 文档日期：2026-07-27

四层详细说明：

1. [应用层：线程、命令与 CoreMark](./01_application_layer.md)
2. [RT-Thread 层：内核、调度与系统服务](./02_rt_thread_layer.md)
3. [BSP 与软件移植层：启动、Trap、驱动和链接](./03_bsp_porting_layer.md)
4. [RTL/SoC 层：CPU、存储器、外设与 FPGA 顶层](./04_rtl_soc_layer.md)
5. [通用 RTL/SoC 架构：从顶层引脚到片上总线](./05_generic_rtl_soc_architecture.md)（不依赖当前工程的架构补充）
6. [SoC 与 RTOS 移植学习指南](./06_soc_os_porting_learning_guide.md)（面向 RTL 背景的快速学习路线与开源项目导读）
7. [RT-Thread 官方源码导读](./07_rt_thread_official_source_guide.md)（官方仓库目录、启动、调度、RV32 port、BSP、构建与 FinSH）
8. [CoreMark 使用、源码结构与 SoC/RT-Thread 接入指南](./08_coremark_usage_structure_and_porting.md)（使用方法、官方源码结构、裸机与 RT-Thread 接入、仿真和有效成绩规则）
9. [HXI 片上互联、BRAM 存储与 Cache 架构说明](./09_hxi_interconnect_bram_cache_architecture.md)（HXI 与 AHB/APB 的关系、多主多从 Crossbar、BRAM 接入、Cache 层次和当前工程演进方案）
10. [CLINT、Machine Timer 与 RT-Thread Tick](./10_clint_machine_timer_and_rtos_tick.md)（CLINT/ACLINT、`mtime/mtimecmp`、当前 Counter 区别、HXI 接入、CPU 中断改造、RV32 访问与验证）
11. [SoC 各组成部分：职责、接口与设计理由](./11_soc_components_roles_interfaces_and_design.md)（CPU、Wrapper、Cache、HXI、地址译码、BRAM、APB、UART、GPIO、Timer、中断、时钟复位与 CDC 的通俗导读）
12. [三人团队的职责边界、交付标准与协作方式](./12_three_person_team_roles_boundaries_and_delivery.md)（队长、SoC RTL Owner、BSP/RT-Thread Owner 的完整责任域、文件所有权、交付物、DoD、Bug 归属、AI 使用与比赛现场分工）
13. [RTL 项目结构规划：Verilator 与 FPGA 友好](./13_rtl_project_structure_for_verilator_and_fpga.md)（推荐目录树、CPU/SoC/FPGA 三类顶层、HXI/Memory/Peripheral 分层、仿真模型与 Xilinx IP 隔离、filelist、构建目标和渐进迁移）
14. [RTL 内部层次与接口规划：从 CPU Core 到 SoC Top](./14_rtl_internal_hierarchy_and_soc_interfaces.md)（`rtl/` 内部目录、CPU Core/Subsystem 边界、I/D HXI Master、IRQ、Crossbar、BRAM、Timer、APB 外设和 SoC 顶层端口）

## 1. 先确定整个系统的层级

最终系统由四层组成：

```text
应用层
├─ main.c
├─ 线程演示
└─ CoreMark

RT-Thread 层
├─ 线程调度
├─ Tick、软件定时器
├─ 信号量、互斥量、邮箱等 IPC
└─ FinSH/MSH 命令行（可选组件）

BSP 和软件移植层
├─ start.S
├─ RISC-V 上下文切换和 Trap 入口
├─ board.c
├─ UART、Timer、IRQ 驱动
├─ rtconfig.h
└─ link.lds

RTL/SoC 层
├─ RV32 CPU
├─ CSR、异常与中断
├─ IROM、DRAM
├─ MMIO 总线
├─ UART
├─ 系统 Tick 定时器
└─ 中断控制器
```

这四层不能倒着做。RT-Thread 假设底层已经具备可用的栈、RAM、Trap、定时器和串口。当前工程最接近“CPU 指令测试和已有 SoC 程序可以运行”的阶段，还没有达到 RTOS 所需的硬件入口条件。

推荐的主线是：

```text
冻结内存和中断接口
    ↓
补齐 SoC/CPU 中断能力
    ↓
裸机 start.S + link.lds + UART
    ↓
裸机定时器中断
    ↓
上下文保存/恢复测试
    ↓
RT-Thread 最小内核
    ↓
FinSH/MSH
    ↓
CoreMark
```

每次修改 CPU 接口、内存映射或 MMIO 时，后续 RTL 仿真、软件 BSP、链接脚本、镜像转换工具和 Vivado 工程都要同步检查。

---

## 2. 当前工程距离运行 RT-Thread 还差什么

根据 `dev-v4.0` 的 RTL，当前基础情况如下。

| 项目 | 当前状态 | RT-Thread 所需状态 |
|---|---|---|
| 指令集 | RV32IM、Zicsr 和部分 Zb | 最小单核 RTOS 基本够用；构建时必须禁用未实现的 C/F/A 扩展 |
| 异常 | 支持非法指令、ECALL、EBREAK、MRET 等基础路径 | 需要继续保留，并与异步中断统一进入 Trap |
| CSR | 有 `mstatus/mtvec/mscratch/mepc/mcause/mtval` | 还需 `mie/mip`，并正确表示中断类型 |
| CPU 中断端口 | `core_top` 和 `myCPU` 没有 IRQ 输入 | 至少增加 Timer IRQ 和 External IRQ |
| `mcause` | 当前 Trap cause 只有 5 位 | 应改为 32 位，`mcause[31]=1` 表示中断 |
| Timer | `counter.sv` 是可启动/停止的性能计数器 | 需要独立的系统 Tick 定时器并产生 IRQ |
| UART | 板级 UART 服务于 `twin_controller` | RT-Thread 需要 CPU 可访问的 MMIO UART |
| IROM | 16 KiB，地址只取 `irom_addr[13:2]` | 带 FinSH 和 CoreMark 时明显偏小，建议扩到 128～256 KiB |
| DRAM | `0x8010_0000..0x8013_ffff`，256 KiB | 容量可以先使用，但要规划静态区、堆和各线程栈 |
| MMIO | SW、KEY、SEG、LED、counter | 还需 UART、Timer、IRQ 控制器 |
| 性能计数 | 有 `cycle/instret` 读接口 | 可用于 CoreMark 计时，但要验证 64 位读取方法 |

以下改造按优先级排列。

---

## 3. RTL 和 SoC 需要进行的改造

### 3.1 第一优先级：给 CPU 增加机器模式中断

#### 3.1.1 CPU 顶层端口

建议给 `core_top.sv` 和 `myCPU.sv` 增加：

```systemverilog
input logic irq_software_i;
input logic irq_timer_i;
input logic irq_external_i;
```

第一版可以只接：

```text
irq_timer_i     ← machine_timer
irq_external_i  ← UART 或简化中断控制器
irq_software_i  ← 0
```

软件中断不是最早的必需项。RT-Thread 的 RV32 common port 在非向量模式下可以使用统一中断入口，先完成定时器和外部中断更重要。

#### 3.1.2 CSR 扩展

`csr_file.sv` 至少增加：

| CSR | 地址 | 本项目需要的位 |
|---|---:|---|
| `mstatus` | `0x300` | `MIE`、`MPIE`、`MPP` |
| `mie` | `0x304` | `MSIE`、`MTIE`、`MEIE` |
| `mtvec` | `0x305` | Direct 模式先行，地址低两位为 0 |
| `mscratch` | `0x340` | RT-Thread common port 用于中断栈切换 |
| `mepc` | `0x341` | 保存异常或中断返回地址 |
| `mcause` | `0x342` | bit31 为中断标志，低位为 cause |
| `mtval` | `0x343` | 异常补充信息；中断时可为 0 |
| `mip` | `0x344` | `MSIP`、`MTIP`、`MEIP` |
| `mhartid` | `0xF14` | 单核固定读 0 |

`mip` 中的硬件 pending 位不应当被普通 CSR 写操作随意覆盖。可按下面方式组织：

```systemverilog
mip.MSIP = irq_software_i;
mip.MTIP = irq_timer_i;
mip.MEIP = irq_external_i;
```

中断有效条件为：

```text
mstatus.MIE = 1
并且
(mie & mip) 中至少有一个对应位为 1
```

机器模式标准 cause 可按下表处理：

| 中断 | `mcause[31]` | `mcause` 低位 |
|---|---:|---:|
| Machine software interrupt | 1 | 3 |
| Machine timer interrupt | 1 | 7 |
| Machine external interrupt | 1 | 11 |

当前 `trap_cause_i` 只有 5 位，不足以表达中断标志。建议把 CSR Trap 接口直接改成：

```systemverilog
input logic [31:0] trap_cause_i;
```

#### 3.1.3 精确中断时机

异步中断不能在流水线任意位置直接改 PC。建议在“架构提交边界”接收中断：

1. 已提交指令的寄存器和存储副作用必须保留；
2. 未提交的年轻指令全部清空；
3. `mepc` 保存下一条尚未执行的架构指令地址；
4. `mcause` 写入中断编号；
5. `mstatus.MPIE ← mstatus.MIE`；
6. `mstatus.MIE ← 0`；
7. PC 重定向到 `mtvec`；
8. `mret` 时恢复 MIE，并跳回 `mepc`。

当前内核是单发射、顺序提交，这比乱序提交处理器容易实现精确中断。但仍需检查 StoreBuffer：

- 中断发生前已经提交的 store 不能丢；
- 未提交 store 不能泄漏到 MMIO；
- 中断入口读写 MMIO 时，不能越过更早的 MMIO 写；
- 如有未完成的 uncached 请求，应等待到边界稳定后再进入 Trap。

建议在 `recovery_ctrl.sv` 增加独立的 `interrupt_take` 路径，而不是把中断伪装成某条普通指令的同步异常。

#### 3.1.4 建议新增的验证

先不要运行 RT-Thread，增加以下 directed tests：

```text
irq_disabled
    mip 有 pending，但 mstatus.MIE=0，不得跳转

irq_masked
    mstatus.MIE=1，但 mie.MTIE=0，不得响应 timer IRQ

timer_irq_entry
    mepc/mcause/mstatus/mtvec 全部符合预期

irq_return
    MRET 后继续执行被打断的程序，通用寄存器不变

irq_during_load
    load 完成边界和 mepc 正确

irq_during_store
    已提交 store 不丢，未提交 store 不发生

irq_priority
    多个 pending 同时出现时优先级确定且可重复
```

### 3.2 第一优先级：新增系统 Tick 定时器

现有 `counter.sv` 用来给比赛程序统计时间，具备 start/stop 控制和跨时钟域处理，但不产生 CPU 中断。它不适合作为 RT-Thread 系统 Tick 的唯一来源。

建议新增：

```text
rtl/soc/machine_timer.sv
```

最容易与 RISC-V 软件适配的模型是 64 位 `mtime/mtimecmp`：

```text
mtime      每个 CPU 时钟或固定参考时钟递增
mtimecmp   软件设定下一次中断时刻
timer_irq  当 mtime >= mtimecmp 时拉高
```

建议在 CPU 时钟域运行，避免第一版引入额外 CDC。寄存器是 32 位总线，因此 64 位值拆成高低两个寄存器：

| 偏移 | 寄存器 | 访问 |
|---:|---|---|
| `0x00` | `MTIME_LO` | 读/写 |
| `0x04` | `MTIME_HI` | 读/写 |
| `0x08` | `MTIMECMP_LO` | 读/写 |
| `0x0C` | `MTIMECMP_HI` | 读/写 |
| `0x10` | `CTRL` | enable 等 |

RT-Thread Tick 设为 1000 Hz 时：

```text
tick_interval = CPU_FREQ_HZ / 1000
next_mtimecmp = current_mtime + tick_interval
```

Timer ISR 中应重新设置下一次 `mtimecmp`，然后调用：

```c
rt_tick_increase();
```

使用 compare 模型时，不需要设计一个含义模糊的“向 pending 位写 1 清除”。只要把 `mtimecmp` 更新到未来，`timer_irq` 就会下降。

### 3.3 第一优先级：增加 CPU 可访问的 MMIO UART

当前 `top.sv` 中的 UART 接在 `twin_controller` 上，波特率为 9600。CPU 既看不到 UART 状态寄存器，也不能直接发送 RT-Thread 日志。因此，现有 UART 链路不能直接充当 RT-Thread console。

建议新增：

```text
rtl/soc/uart_mmio.sv
```

可采用以下简化寄存器：

| 偏移 | 寄存器 | 说明 |
|---:|---|---|
| `0x00` | `TXDATA` | 写低 8 位开始发送 |
| `0x04` | `RXDATA` | 读低 8 位；可在读取后清 RX valid |
| `0x08` | `STATUS` | bit0 TX busy，bit1 RX valid |
| `0x0C` | `CTRL` | TX/RX/中断使能 |
| `0x10` | `IRQ_STATUS` | RX pending、TX empty 等 |
| `0x14` | `BAUD_DIV` | 可选；也可综合时固定 |

实施顺序：

1. 先完成 polling TX，让 `rt_kprintf()` 能输出；
2. 再完成 polling RX，让 `rt_hw_console_getchar()` 能返回字符；
3. 最后增加 RX 中断和小型 FIFO，支持稳定的 FinSH 输入。

需要提前决定板级物理串口如何分配：

- 如果开发板有第二路 UART，把 RT-Thread console 放到第二路最简单；
- 如果只有一路 UART，可在 `top.sv` 中增加模式选择，在 digital-twin 与 console 之间切换；
- 也可扩展 digital-twin 协议，把 console 数据作为一种报文传输，但软件和上位机都会更复杂。

不建议让 `twin_controller` 和 RT-Thread UART 同时直接驱动同一个 TX 引脚。

### 3.4 第二优先级：中断控制器

最小系统只有两个硬件中断：

```text
Machine timer interrupt  → mcause 7
UART external interrupt  → mcause 11
```

第一阶段可以让 UART 直接产生 `irq_external_i`。进入外部中断后，软件读取 UART `IRQ_STATUS` 判断并处理。

当需要 UART、GPIO、按键等多个外部中断时，增加：

```text
rtl/soc/irq_ctrl.sv
```

至少提供：

- pending；
- enable；
- 可选 priority；
- claim；
- complete。

这可以做成简化 PLIC，也可以直接实现标准 PLIC 的一小部分。比赛移植并不要求完整复制大型 PLIC。驱动和文档必须明确采用哪一种接口。

### 3.5 第一优先级：重新规划 IROM、DRAM 和启动镜像

#### 3.5.1 扩大 IROM

当前 IROM 为 4096 words，即 16 KiB。RT-Thread 最小内核也许能通过激进裁剪勉强放入，但加上 FinSH、驱动、字符串和 CoreMark 后很容易溢出。

建议：

```text
第一阶段：IROM 扩到 128 KiB
稳妥方案：IROM 扩到 256 KiB
DRAM：继续使用现有 256 KiB
```

如果扩到 256 KiB，IROM 共有 65536 words，地址索引应由：

```systemverilog
irom_addr[13:2]
```

调整为：

```systemverilog
irom_addr[17:2]
```

同时更新：

- `IROM_0` IP 深度；
- Verilator IROM 模型；
- Vivado Tcl；
- 镜像长度检查；
- 软件内存映射文档。

#### 3.5.2 哈佛结构下的关键限制

当前 CPU 的取指口只访问 IROM，数据口只访问 DRAM/MMIO。C 程序中的以下内容都是通过 load 指令读取的：

- 字符串常量；
- `const` 数组；
- 函数指针表；
- RT-Thread 自动初始化表 `.rti_fn*`；
- FinSH 命令表 `FSymTab/VSymTab`；
- CoreMark 测试数据。

因此，不能简单地把 `.rodata`、`FSymTab` 和 `.rti_fn*` 跟 `.text` 一起放进“只能取指”的 IROM。否则程序能取到指令，却在打印第一个字符串或扫描初始化表时读到错误数据。

有两种方案。

**方案 A：维持严格哈佛结构**

```text
IROM：只放 .start 和 .text
DRAM：放 .rodata、RT-Thread 表、.data、.bss、heap、stack
```

构建工具从一个 ELF 分别生成：

```text
firmware_imem.mem
firmware_dmem.mem
```

这是当前工程最容易落地的方案。

**方案 B：让数据口也能读取 ROM**

在 `SocMemBridge` 中增加 IROM 的数据读窗口，或改成统一存储器。此时可采用常见的 `.text/.rodata` 全部在 ROM、`.data` 启动时从 ROM 复制到 RAM 的方案。

方案 B 更接近传统 MCU，但会改动存储接口和 IP 端口。比赛周期有限时，建议先完成方案 A。

#### 3.5.3 建议的内存映射

| 地址范围 | 用途 |
|---|---|
| `0x8000_0000..0x8003_FFFF` | IROM，256 KiB，只放可执行指令 |
| `0x8010_0000..0x8013_FFFF` | DRAM，256 KiB |
| `0x8020_0000..0x8020_00FF` | 现有 SW/KEY/SEG/LED/counter |
| `0x8020_1000..0x8020_10FF` | MMIO UART |
| `0x8020_2000..0x8020_20FF` | Machine timer |
| `0x8020_3000..0x8020_30FF` | 简化 IRQ controller |

地址可以调整，但 RTL、链接脚本、驱动头文件、仿真模型和文档必须只有一份权威定义。

建议增加：

```text
rtl/soc/soc_addr_pkg.sv
sw/bsp/superscalar/include/soc.h
docs/design/soc_memory_map.md
```

三处内容由同一张地址表维护，避免 RTL 和 C 代码各自手写后漂移。

### 3.6 MMIO、Cache 和 StoreBuffer 的约束

当前 core 只把 DRAM 地址判断为 cacheable，其他地址走 uncached，这个方向是正确的。为了运行 OS，还要明确：

- UART、Timer、IRQ 寄存器必须 uncached；
- MMIO 访问不能被合并；
- MMIO store 不能越过前面的 MMIO store；
- 读取状态寄存器前，前面的控制写必须已经生效；
- 外设读必须遵守 `req_valid/req_ready/resp_valid`；
- 未命中的非法 MMIO 地址应返回确定值，最好留出 bus error 统计；
- `FENCE` 至少需要保证所有更早的 uncached/内存请求完成。

如果 StoreBuffer 会接收 uncached store，建议单独处理：MMIO store 在提交点发出并等待接受，不进入普通可合并的缓存 store 流程。

### 3.7 可后做的 CPU 功能

以下内容有用，但不是最早的阻塞项：

- `WFI`：可让 idle thread 降低动态功耗；未实现时可让 idle hook 执行普通循环；
- 软件中断 MSIP；
- 完整 PLIC；
- UART FIFO 和 DMA；
- 动态 Bootloader；
- PMP、用户模式、MMU；
- 文件系统和动态模块加载。

RT-Thread Nano 或单核 RT-Thread 内核不要求 MMU，也不要求 Linux 式进程。

### 3.8 RTL 建议新增或修改的文件

```text
rtl/
├─ core/
│  ├─ core_top.sv                    # 增加 IRQ 输入和 interrupt_take
│  ├─ myCPU.sv                       # 透传 IRQ
│  ├─ commit/
│  │  └─ csr_file.sv                 # mie/mip/mcause 中断位
│  └─ control/
│     └─ recovery_ctrl.sv            # 精确中断重定向
│
└─ soc/
   ├─ student_top.sv                 # 连接 Timer/UART/IRQ
   ├─ SocMemBridge.sv                # 新增 MMIO 地址译码
   ├─ soc_addr_pkg.sv                # 地址常量
   ├─ machine_timer.sv               # 新增
   ├─ uart_mmio.sv                   # 新增
   ├─ irq_ctrl.sv                    # 第二阶段新增
   └─ top.sv                         # 处理 physical UART 归属
```

---

## 4. RT-Thread 开源项目里有什么

RT-Thread 是一个开源 RTOS。它提供线程调度、Tick、软件定时器、IPC、内存管理等通用内核功能，但不认识本项目的 UART 地址、定时器结构、FPGA 存储器和复位入口。

截至本文编写时，RT-Thread 官方最新稳定发布为 v5.2.2。去年的优秀作品使用 RT-Thread Nano 3.1.5。对本项目更合理的做法是：

1. 固定一个明确 tag，不直接追踪 `master`；
2. 优先尝试当前稳定版的单核最小配置；
3. 关闭 SMP、MMU、FPU、文件系统、网络、动态模块等无关组件；
4. 如果当前版本 common port 与自研 CPU 适配工作量超出赛程，再以去年使用的 Nano 版本作为备选，而不是混用两套版本的 port 文件。

### 4.1 官方仓库顶层目录

RT-Thread 仓库主要目录如下：

| 目录 | 用途 | 本项目是否需要 |
|---|---|---|
| `src/` | 调度器、线程、时钟、IPC、内存等内核 | 需要 |
| `include/` | `rtthread.h`、`rthw.h` 等内核头文件 | 需要 |
| `libcpu/` | 各 CPU 架构的上下文切换和底层 port | 需要 RISC-V RV32 部分 |
| `components/finsh/` | FinSH/MSH 串口命令行 | 第二阶段需要 |
| `components/drivers/` | 通用设备框架 | 可选；最小移植可先不用 |
| `bsp/` | 各芯片/板卡的参考 BSP | 用于参考结构，不可直接照搬地址 |
| `examples/` | 示例 | 选读 |
| `tools/` | SCons、配置等工具 | 使用官方构建系统时需要 |

`src/` 中与最小单核内核关系最直接的文件包括：

```text
clock.c
components.c
cpu_up.c
idle.c
ipc.c
irq.c
kservice.c
mem.c
object.c
scheduler_comm.c
scheduler_up.c
thread.c
timer.c
```

实际选择应交给 RT-Thread 自己的 `SConscript/Kconfig/rtconfig.h`，不建议手工复制几个 `.c` 后长期脱离官方构建关系。

### 4.2 RV32 common port 中有用的文件

官方 `libcpu/risc-v/common/` 当前包含：

| 文件 | 作用 | 本项目处理方式 |
|---|---|---|
| `context_gcc.S` | 线程上下文切换、开关中断等 | 重点复用并逐条核对所需指令 |
| `interrupt_gcc.S` | 中断入口、寄存器保存恢复、IRQ 返回 | 重点复用 |
| `cpuport.c` | 初始线程栈等 CPU port 逻辑 | 复用并检查配置 |
| `cpuport.h` | RV32/RV64 字长和 load/store 宏 | 复用 |
| `riscv-ops.h` | CSR 读写操作 | 复用 |
| `rt_hw_stack_frame.h` | Trap/线程栈帧格式 | 复用 |
| `trap_common.c` | 非向量模式中断注册和分发 | 可复用 |
| `atomic_riscv.c` | RISC-V 原子操作适配 | 必须检查是否产生 CPU 未实现的 A 扩展指令 |

本项目没有 RVC，编译参数不能包含 `c`；没有浮点上下文时，应关闭 `ARCH_RISCV_FPU`；单核项目关闭 SMP。还要检查 `atomic_riscv.c` 和配置是否依赖 `amo*` 指令。若最小单核配置仍生成 A 扩展指令，可以：

- 使用关中断实现的单核临界区；
- 选用不依赖 A 扩展的 port 配置；
- 或最后再为 CPU 实现 A 扩展。

不建议为移植第一个版本立刻增加完整 RV32A。

### 4.3 哪些 BSP 文件必须由本项目提供

建议在 RT-Thread 仓库内新增：

```text
bsp/superscalar/
├─ applications/
│  ├─ main.c
│  └─ coremark_cmd.c
├─ board/
│  ├─ board.c
│  ├─ board.h
│  ├─ start.S
│  ├─ trap_gcc.S
│  └─ soc.h
├─ drivers/
│  ├─ drv_uart.c
│  ├─ drv_timer.c
│  └─ drv_irq.c
├─ Kconfig
├─ SConscript
├─ SConstruct
├─ rtconfig.h
├─ rtconfig.py
└─ link.lds
```

也可以把官方仓库作为 `sw/rt-thread/`，把 BSP 放在：

```text
sw/bsp/superscalar/
```

两种结构都可以。比赛工程更适合把 RT-Thread 固定在 `sw/third_party/rt-thread/` 或 git submodule 中，本项目 BSP 独立放在 `sw/bsp/superscalar/`，这样第三方源码和自研代码边界清楚。

### 4.4 最小 BSP 需要实现的接口

至少需要：

```c
void rt_hw_board_init(void);
void rt_hw_console_output(const char *str);
char rt_hw_console_getchar(void);

rt_base_t rt_hw_interrupt_disable(void);
void rt_hw_interrupt_enable(rt_base_t level);

void rt_hw_context_switch(...);
void rt_hw_context_switch_to(...);
void rt_hw_context_switch_interrupt(...);

rt_uint8_t *rt_hw_stack_init(...);
```

另外需要：

- `start.S` 设置 `sp/gp/mscratch/mtvec`；
- Timer ISR 调用 `rt_tick_increase()`；
- `rt_hw_board_init()` 初始化 UART、Timer、IRQ 和 heap；
- 如果采用 `trap_common.c`，实现 `rt_hw_do_after_save_above()` 适配统一入口；
- 在启用任何外设中断前注册对应 handler。

### 4.5 推荐的移植顺序

#### 步骤 1：裸机 BSP

先只编译：

```text
start.S + board.c + drv_uart.c + main.c
```

目标输出：

```text
hello from superscalar
```

此时验证链接布局、栈、全局变量、字符串读取和 UART。

#### 步骤 2：裸机 Trap

设置 `mtvec`，执行 ECALL，验证：

```text
进入 Trap → 打印 mcause/mepc → 修改 mepc → mret
```

#### 步骤 3：裸机 Timer IRQ

打开 `mstatus.MIE` 和 `mie.MTIE`，每 1 ms 产生 Timer IRQ。先只累加一个全局计数器，不启动 RT-Thread。

#### 步骤 4：接入 RT-Thread 内核和 RISC-V common port

只启用：

- 单核 UP；
- user main；
- thread；
- timer；
- semaphore；
- heap（如需要动态创建线程）。

创建两个静态线程，验证时间片和优先级。

#### 步骤 5：接入 FinSH

启用 `components/finsh/`，实现 console 输入输出。链接脚本必须保留 `FSymTab/VSymTab`。

#### 步骤 6：接入 CoreMark

CoreMark 作为应用或 MSH 命令加入，不修改 RT-Thread 内核。

---

## 5. 除 RT-Thread 源码外，哪些文件要自己写

“链接脚本”“启动脚本”“构建脚本”经常被混为一类，实际职责不同。

| 文件 | 所属层 | 作用 | 是否由本项目配置 |
|---|---|---|---|
| `link.lds` | 软件链接 | 决定代码、数据、堆栈地址 | 必须自己写 |
| `start.S` | 启动运行时 | 复位后初始化 `sp/gp/bss/mtvec` | 必须适配 |
| `trap_gcc.S` | 架构移植 | 对接 RT-Thread common trap | 必须适配 |
| `rtconfig.h` | RT-Thread 配置 | 选择内核组件、栈、优先级、Tick | 必须配置 |
| `SConstruct/SConscript` 或 `Makefile` | 构建 | 选择源码和编译参数 | 必须配置 |
| `rtconfig.py` | 工具链 | GCC 前缀、参数、链接选项 | 必须配置 |
| `elf2mem.py` | 镜像转换 | 从 ELF 分离 IMEM/DMEM | 本项目需要新增 |
| `create_vivado_project.tcl` | FPGA 工程 | 创建 IP、加入 COE/MEM、生成 bitstream | 需要修改 |
| RTL filelist `.f` | RTL 仿真 | 加入新 Timer/UART/IRQ 模块 | 需要修改 |

`link.lds` 不属于 RTL，也不会放进 Vivado 综合。它只由 RISC-V GCC/LD 使用。

### 5.1 适合当前哈佛结构的链接脚本骨架

下面是设计参考，不是可以不经验证直接提交的最终版：

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

    /*
     * 当前数据口不能读取 IROM，所以 rodata 和 RT-Thread 的表放 DRAM。
     * elf2mem 工具负责把这些 section 的初值写入 DMEM 镜像。
     */
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

        . = ALIGN(8);
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
        . = ALIGN(8);
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

    ASSERT(__heap_start < __heap_end, "DRAM overflow: no heap/stack space")

    /DISCARD/ :
    {
        *(.comment)
        *(.note*)
    }
}
```

这里没有使用：

```ld
.data > DRAM AT > IROM
```

原因是当前 CPU 数据口不能从 IROM 复制 `.data`。构建脚本应按 section 的运行地址直接生成 DRAM 初始镜像。

不要简单执行一次 `objcopy -O binary` 得到一个跨越 IROM 和 DRAM 的大平面文件。两个区域相隔约 1 MiB，会产生大量空洞，也无法直接表达两个独立 BRAM。应根据 ELF section 地址分别生成两个镜像。

### 5.2 `start.S` 需要做什么

第一版启动流程可以是：

```asm
    .section .start, "ax"
    .globl _start

_start:
    csrci mstatus, 0x8          /* 启动阶段先关全局中断 */

    la sp, __boot_stack_top
    la gp, __global_pointer$

    la t0, __irq_stack_top
    csrw mscratch, t0

    /* 清零 BSS */
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

    csrw mie, zero
    call entry

3:
    j 3b
```

注意：

- `.data/.rodata` 已由 DMEM 镜像预初始化时，启动代码不用从 IROM 复制；
- 启动阶段不要过早打开中断；
- `mscratch` 的用法必须与采用的 RT-Thread `interrupt_gcc.S` 一致；
- 栈按 RISC-V ABI 至少 16 字节对齐；
- `entry` 的实际名字由 BSP 启动组织决定，也可能调用 `rtthread_startup()`。

### 5.3 推荐编译参数

根据当前 CPU 能力，先采用：

```text
-march=rv32im_zicsr
-mabi=ilp32
-mcmodel=medany
-ffunction-sections
-fdata-sections
-fno-common
-O2
```

链接参数：

```text
-nostartfiles
-Wl,-T,link.lds
-Wl,-Map,firmware.map
-Wl,--gc-sections
```

不要使用：

```text
-march=rv32imac
```

因为当前 CPU 没有 C 压缩指令和 A 原子指令。若工具链把 Zicsr 视为独立扩展，应保留 `_zicsr`；最终以反汇编扫描为准。

### 5.4 `elf2mem.py` 应做什么

建议工具直接读取 ELF section：

```text
可执行 section，地址位于 IROM → imem.mem/imem.coe
可加载数据 section，地址位于 DRAM → dmem.mem/dmem.coe
NOBITS 的 .bss → 不写镜像，由 start.S 清零
其他地址 → 报错
```

工具还应检查：

- IROM/DRAM 是否越界；
- section 是否重叠；
- 入口是否为 `0x8000_0000`；
- 地址是否至少按字节正确落位；
- 小端字节如何组合成 32 位 word；
- 输出一份 section 布局摘要。

构建后至少保存：

```text
firmware.elf
firmware.map
firmware.dis
firmware.sections.txt
firmware_imem.mem
firmware_dmem.mem
```

---

## 6. CoreMark 是什么，接在哪一层

本节把问题中的“com”按上下文理解为 **CoreMark**。

CoreMark 是 EEMBC 发布的 CPU/MCU 基准程序。它使用链表、矩阵运算、状态机和 CRC 校验测试处理器的整数执行能力。它不是操作系统，也不是 RTL 模块。

它位于应用层：

```text
CoreMark
   ↓ 调用计时、打印、内存等移植接口
RT-Thread（可选）
   ↓
BSP/驱动
   ↓
CPU 与 SoC RTL
```

CoreMark 有两种运行方式。

### 6.1 裸机运行

```text
start.S → CoreMark main → UART 输出结果
```

优点：

- OS 干扰少；
- 更接近 CPU 核心性能；
- 适合做纯处理器优化对比。

缺点：

- 不能同时证明 RT-Thread 移植成功；
- 需要自己的计时和打印接口。

### 6.2 在 RT-Thread 中运行

```text
RT-Thread 启动
    ↓
FinSH 输入 coremark
    ↓
创建 CoreMark 线程
    ↓
线程执行 CoreMark
    ↓
UART 输出 CRC、时间和分数
```

这正是去年优秀作品采用的展示方式。CoreMark 源码仍然在编译时静态链接进同一个 `firmware.elf`。FinSH 命令只负责调用已经存在的函数，不会从电脑加载一个新的可执行文件。

建议同时保留两套结果：

```text
Bare-metal CoreMark：代表 CPU/SoC 的干净性能
RT-Thread CoreMark：代表操作系统环境下的实际运行结果
```

报告中不要把两者混成同一组成绩。

### 6.3 CoreMark 哪些文件需要编译

官方核心源码：

```text
core_list_join.c
core_main.c
core_matrix.c
core_state.c
core_util.c
coremark.h
```

平台移植文件：

```text
core_portme.c
core_portme.h
core_portme.mak
```

真正需要按本项目修改的是 `core_portme*`：

- `start_time()`；
- `stop_time()`；
- `get_time()`；
- `time_in_secs()`；
- `portable_init()`；
- `portable_fini()`；
- `ee_printf()`；
- 数据类型和内存分配方式。

RT-Thread 下可以把 CoreMark 的 `main` 改名：

```text
-Dmain=coremark_main
```

再写一个命令包装：

```c
static void coremark_thread_entry(void *parameter)
{
    coremark_main();
}

static int coremark_cmd(void)
{
    /* 创建或启动静态 CoreMark 线程 */
    return 0;
}

MSH_CMD_EXPORT(coremark_cmd, run CoreMark benchmark);
```

### 6.4 CoreMark 如何计时

当前 CPU 已有 `cycle/cycleh`，可以读取 64 位周期数。RV32 读取时不能简单地先读低 32 位再读高 32 位，因为低位可能在两次 CSR 读取之间溢出。

应采用：

```text
读取 cycleh → 读取 cycle → 再读取 cycleh
如果两次 cycleh 不相等，重新读取
```

运行时间：

```text
seconds = (end_cycle - start_cycle) / CPU_FREQ_HZ
```

CoreMark：

```text
CoreMark score = iterations / seconds
CoreMark/MHz   = CoreMark score / CPU_FREQ_MHz
```

需要确保报告中的 CPU 频率是真实实现后的频率，而不是仅使用 PLL 参数或目标约束。

### 6.5 CoreMark 成绩的基本规则

按官方仓库要求：

- 正式结果运行时间至少 10 秒；
- validation CRC 必须通过；
- 所有核心源码使用相同编译参数；
- 不允许修改 `core_portme*` 以外的 benchmark 核心源码；
- 可以修改迭代次数、计时方式、内存获取方式和平台移植文件。

不要只打印一个很高的数字。报告应包含：

```text
CPU 频率
编译器版本
-march/-mabi/-O 选项
RT-Thread 或 bare-metal 模式
迭代次数
运行时间
CRC 是否通过
CoreMark
CoreMark/MHz
```

去年作品在 CoreMark 期间关闭全局中断，以减少系统 Tick 干扰。这样做会让 RT-Thread 在测试期间失去正常调度和 Tick，不适合证明“操作系统负载下的性能”。更清楚的做法是：

- 裸机模式给出纯 CPU 分数；
- RT-Thread 模式保留中断，CoreMark 线程设为较高优先级，明确报告这是 OS 环境成绩；
- 如额外测试关中断模式，单独标注，不与标准成绩混在一起。

---

## 7. 建议的软件目录

```text
superScalar/
├─ rtl/
│  ├─ core/
│  └─ soc/
│
├─ sw/
│  ├─ third_party/
│  │  ├─ rt-thread/                 # 固定 tag
│  │  └─ coremark/                  # 固定官方版本
│  │
│  ├─ bsp/
│  │  └─ superscalar/
│  │     ├─ applications/
│  │     │  ├─ main.c
│  │     │  ├─ thread_demo.c
│  │     │  └─ coremark_cmd.c
│  │     ├─ board/
│  │     │  ├─ board.c
│  │     │  ├─ board.h
│  │     │  ├─ start.S
│  │     │  ├─ trap_gcc.S
│  │     │  └─ soc.h
│  │     ├─ drivers/
│  │     │  ├─ drv_uart.c
│  │     │  ├─ drv_timer.c
│  │     │  └─ drv_irq.c
│  │     ├─ link.lds
│  │     ├─ rtconfig.h
│  │     ├─ rtconfig.py
│  │     ├─ Kconfig
│  │     ├─ SConstruct
│  │     └─ SConscript
│  │
│  └─ tools/
│     ├─ elf2mem.py
│     ├─ check_isa.py
│     └─ update_bitstream.tcl
│
├─ build/
│  └─ software/
│     ├─ firmware.elf
│     ├─ firmware.map
│     ├─ firmware.dis
│     ├─ firmware_imem.mem
│     └─ firmware_dmem.mem
│
└─ docs/
   └─ design/
      ├─ soc_memory_map.md
      ├─ interrupt_contract.md
      └─ rt_thread_port.md
```

`data/` 继续保存现有 RV32/SRC 测试镜像，不要把 RT-Thread 源码和 BSP 混入其中。

---

## 8. 分阶段开发和验收

| 阶段 | 主要工作 | 完成标准 |
|---|---|---|
| 0 | 冻结内存映射、中断 cause、UART/Timer 寄存器 | 一份 RTL/C/文档共用的接口定义 |
| 1 | 扩 IROM、分离 IMEM/DMEM 镜像 | 裸机 C 能打印字符串，全局变量正确 |
| 2 | CSR 和精确中断 | ECALL、Timer IRQ、MRET directed tests 全通过 |
| 3 | MMIO UART | polling TX/RX 在 Verilator 和 FPGA 都通过 |
| 4 | machine timer | 1 ms Tick 长时间稳定，无丢中断 |
| 5 | RT-Thread 最小内核 | 两个线程可抢占/延时，栈和寄存器不损坏 |
| 6 | FinSH | 出现 `msh >`，可输入 `help/ps` |
| 7 | CoreMark | CRC 通过，裸机和 RTOS 两套结果可复现 |
| 8 | FPGA 自动化 | 一条命令构建 ELF、生成 MEM、更新 bitstream、下载 |

每个阶段都同时保留 Verilator 和 FPGA 证据。不要在 FPGA 上看到一次输出就跳过回归。

### 8.1 建议增加的自动测试

```text
make sim-baremetal TEST=hello
make sim-baremetal TEST=bss_data
make sim-baremetal TEST=ecall_mret
make sim-baremetal TEST=timer_irq
make sim-baremetal TEST=uart_loopback
make sim-rtthread TEST=boot
make sim-rtthread TEST=thread_switch
make sim-rtthread TEST=finsh
make sim-coremark MODE=baremetal
make sim-coremark MODE=rtthread
```

UART 日志可做成自检：

```text
看到 "RT-Thread"                    → boot 通过
看到 "thread A"/"thread B" 交替     → 调度通过
看到 "msh >"                        → FinSH 通过
看到 "Correct operation validated"  → CoreMark 通过
```

---

## 9. 三人分工建议

### 队长：硬件契约和系统集成

- CPU IRQ、CSR、精确 Trap；
- 内存映射和 SoC 总线；
- RTL/BSP 接口评审；
- 主分支、每日可启动版本；
- FPGA 集成和最终答辩主线。

### 队员 2：RT-Thread 和 BSP

- `start.S/link.lds/trap_gcc.S`；
- `rtconfig.h`；
- UART、Timer、IRQ 驱动；
- RT-Thread common port 适配；
- FinSH 和 CoreMark 命令。

### 队员 3：构建、镜像和验证

- `elf2mem.py`；
- 编译、反汇编和 ISA 检查；
- Verilator 外设模型；
- 中断/上下文/RT-Thread 自动测试；
- CoreMark 结果记录；
- Vivado MEM/bitstream 更新脚本。

队长需要跨两条边界负责：

```text
CPU ↔ SoC
SoC ↔ BSP
```

这两处一旦无人统一，最常见的问题就是 RTL 地址、C 宏和链接脚本各不相同。

---

## 10. 最重要的实施决定

开始编码前应书面确定：

1. IROM 最终扩到 128 KiB 还是 256 KiB；
2. `.rodata` 放 DRAM，还是让数据口能读取 IROM；
3. RT-Thread console 是否占用现有 physical UART；
4. 第一版外部中断采用 UART 直连，还是简化 PLIC；
5. 采用 RT-Thread 哪个固定 tag；
6. 使用 SCons 还是项目现有 Makefile 统一构建；
7. CoreMark 是否同时保留 bare-metal 和 RTOS 两个 profile。

按当前工程风险，推荐答案是：

```text
IROM 256 KiB
.text 放 IROM，其余可读数据放 DRAM
新增 CPU 可见 MMIO UART
Timer IRQ + UART 直连外部 IRQ 先行
固定 RT-Thread v5.2.2，最小单核配置
保留 SCons BSP，同时由顶层 Makefile 调用
CoreMark 同时保留 bare-metal/RT-Thread 两套结果
```

---

## 11. 参考资料

- [RT-Thread 官方仓库](https://github.com/RT-Thread/rt-thread)
- [RT-Thread v5.2.2 Releases](https://github.com/RT-Thread/rt-thread/releases)
- [RT-Thread RV32 common port 与移植说明](https://github.com/RT-Thread/rt-thread/tree/master/libcpu/risc-v/common)
- [RT-Thread 内核源码目录](https://github.com/RT-Thread/rt-thread/tree/master/src)
- [RT-Thread FinSH 组件](https://github.com/RT-Thread/rt-thread/tree/master/components/finsh)
- [RT-Thread RISC-V BSP 链接脚本参考](https://github.com/RT-Thread/rt-thread/blob/master/bsp/qemu-virt64-riscv/link.lds)
- [EEMBC CoreMark 官方仓库](https://github.com/eembc/coremark)
- [CoreMark bare-metal 移植说明](https://github.com/eembc/coremark/blob/main/barebones_porting.md)

参考官方 BSP 时只借鉴目录、section 和接口组织。QEMU RISC-V BSP 可能是 RV64、统一内存、MMU 或不同中断控制器，不能直接复制其地址和编译宏到本项目。
