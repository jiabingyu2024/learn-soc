# 通用 RTL/SoC 架构：从顶层引脚到片上总线

> 本文介绍一种常见、可迁移的 CPU/SoC 项目组织方式。它不以某一份现有 RTL 为标准答案，也不要求必须使用某一种 CPU 核或总线协议。  
> 与当前项目相关的具体修改仍可参考：[04 RTL/SoC 层](./04_rtl_soc_layer.md)。

## 1. 先建立一张完整的 SoC 图

一个可以运行软件的 SoC，至少要同时解决六件事：

1. CPU 从哪里取第一条指令；
2. 指令和数据访问怎样到达存储器；
3. UART、GPIO、Timer 等外设怎样被 CPU 读写；
4. 外设怎样向 CPU 发出中断；
5. 时钟、复位怎样送到每个模块；
6. 仿真模型和 FPGA 上的真实 IP 怎样保持相同的接口与行为。

一个常见的单核 MCU 型 SoC 可以画成：

```text
                         ┌────────────────────────────┐
外部时钟、复位 ─────────▶│ Clock / Reset Controller   │
                         └──────────────┬─────────────┘
                                        │ clk, reset
                                        ▼
┌──────────────────────────────────────────────────────────────────┐
│                             soc_top                              │
│                                                                  │
│  ┌────────────┐   I-Bus    ┌──────────────────────────────────┐  │
│  │            ├───────────▶│                                  │  │
│  │  CPU Core  │            │       System Interconnect        │  │
│  │            ├───────────▶│   地址译码、仲裁、返回通路、报错   │  │
│  └─────▲──────┘   D-Bus    └───┬─────────┬───────────┬───────┘  │
│        │ IRQ                    │         │           │          │
│        │                        ▼         ▼           ▼          │
│  ┌─────┴────────┐         ┌────────┐ ┌────────┐ ┌───────────┐   │
│  │ Interrupt    │◀────────│ Timer  │ │ Boot   │ │ SRAM /    │   │
│  │ Controller   │◀────────│ UART   │ │ ROM    │ │ DDR Ctrl  │   │
│  └──────────────┘         └────┬───┘ └────────┘ └───────────┘   │
│                                │                                 │
│                                ▼                                 │
│                         ┌───────────────┐                         │
│                         │ APB/外设子系统 │                         │
│                         │ GPIO SPI I2C  │                         │
│                         │ PWM WDT ...   │                         │
│                         └───────────────┘                         │
└──────────────────────────────────────────────────────────────────┘
       │          │          │          │                 │
       ▼          ▼          ▼          ▼                 ▼
     UART       GPIO        SPI        JTAG            外部存储器
     引脚        引脚        引脚        引脚            或 FPGA DDR
```

图中最容易混淆的是“接口”一词。它可能指三类不同的东西：

| 接口种类 | 位于哪里 | 例子 |
| --- | --- | --- |
| 板级物理接口 | FPGA/芯片顶层与外界之间 | 时钟、复位、UART_RX、UART_TX、GPIO、DDR 引脚 |
| 片上模块接口 | RTL 模块之间 | CPU 指令口、数据口、AXI、AHB、APB、中断线 |
| 软件可见接口 | CPU 通过地址访问 | GPIO 寄存器、UART 状态寄存器、Timer 比较值 |

设计 SoC 时要把这三类接口分别写清楚。否则会出现“RTL 有 UART 引脚，但软件找不到 UART 寄存器”或者“软件配置了 Timer，中断却没有接到 CPU”这样的断链。

## 2. 两个顶层：板级顶层和 SoC 顶层

常见 FPGA 项目不会把所有内容都塞进一个 `top.sv`。较稳妥的组织是保留两个顶层。

### 2.1 `fpga_top`：面向开发板

`fpga_top` 的端口与开发板原理图和 XDC/QSF 约束一一对应。它主要放置：

- 差分时钟输入缓冲；
- PLL/MMCM；
- 外部复位同步；
- FPGA 厂商提供的 DDR、BRAM、JTAG 等 IP；
- 三态 I/O 缓冲；
- 与具体开发板有关的引脚复用；
- `soc_top` 实例。

下面是一份示意端口，不代表所有项目都必须具备这些引脚：

```systemverilog
module fpga_top (
    // 时钟与复位
    input  logic        sys_clk_p,
    input  logic        sys_clk_n,
    input  logic        ext_reset_n,

    // 串口
    input  logic        uart_rx_i,
    output logic        uart_tx_o,

    // SPI / QSPI Flash
    output logic        flash_sck_o,
    output logic        flash_cs_no,
    inout  wire  [3:0]  flash_dq_io,

    // GPIO
    inout  wire  [31:0] gpio_io,

    // 调试
    input  logic        jtag_tck_i,
    input  logic        jtag_tms_i,
    input  logic        jtag_tdi_i,
    output logic        jtag_tdo_o,

    // 板上显示
    output logic [7:0]  led_o
);
```

这里的 `inout` 一般只停留在板级顶层。进入 SoC 内部后，把双向 GPIO 拆成三组单向信号：

```text
gpio_i   ：从引脚读回的电平
gpio_o   ：准备输出到引脚的电平
gpio_oe  ：输出使能，1 表示由 SoC 驱动，0 表示高阻输入
```

FPGA 的 I/O Buffer 在 `fpga_top` 里根据这三组信号生成真正的三态引脚。这样既便于仿真，也能避免在内部层次大量使用 `inout`。

### 2.2 `soc_top`：面向逻辑系统

`soc_top` 尽量不包含开发板专属原语。它接收已经处理好的单端系统时钟和复位，内部实例化 CPU、总线、存储器接口、外设和中断控制器。其端口可以写成：

```systemverilog
module soc_top #(
    parameter logic [31:0] RESET_VECTOR = 32'h0000_0000,
    parameter int unsigned GPIO_WIDTH   = 32
) (
    input  logic                  clk_i,
    input  logic                  rst_ni,

    // UART
    input  logic                  uart_rx_i,
    output logic                  uart_tx_o,

    // GPIO：内部不使用 inout
    input  logic [GPIO_WIDTH-1:0] gpio_i,
    output logic [GPIO_WIDTH-1:0] gpio_o,
    output logic [GPIO_WIDTH-1:0] gpio_oe_o,

    // 可选的调试接口
    input  logic                  jtag_tck_i,
    input  logic                  jtag_tms_i,
    input  logic                  jtag_tdi_i,
    output logic                  jtag_tdo_o
);
```

如果 DDR 控制器必须放在板级顶层，`soc_top` 不直接连接 DDR 物理引脚，而是暴露一组 AXI 主设备接口，与 DDR 控制器的 AXI 从设备口相连。这也是“板级顶层”和“逻辑 SoC 顶层”分开的实际价值：更换开发板时，CPU、总线、寄存器地址和软件大多不用改变。

## 3. SoC 内部怎样分层

通用的层次组织可以参考下面的结构：

```text
soc_top
├─ clock_reset_subsystem
│  ├─ reset_sync
│  ├─ clock_divider
│  └─ system_control
│
├─ cpu_subsystem
│  ├─ cpu_core                 CPU 核本体
│  ├─ cpu_wrapper              接口极性、宽度和协议适配
│  ├─ instruction_cache        可选
│  ├─ data_cache               可选
│  └─ debug_module             可选
│
├─ bus_subsystem
│  ├─ instruction_interconnect
│  ├─ data/system_interconnect
│  ├─ address_decoder
│  ├─ arbiter
│  ├─ default_error_slave
│  └─ ahb_to_apb / axi_to_apb  可选
│
├─ memory_subsystem
│  ├─ boot_rom
│  ├─ on_chip_sram
│  ├─ memory_arbiter
│  └─ external_memory_ctrl     可选
│
├─ peripheral_subsystem
│  ├─ gpio
│  ├─ uart
│  ├─ timer
│  ├─ spi / i2c / pwm
│  └─ watchdog
│
├─ interrupt_subsystem
│  ├─ local_timer_irq
│  ├─ software_irq
│  └─ external_irq_controller
│
└─ debug_and_observation
   ├─ jtag_debug
   ├─ trace
   └─ performance_counters
```

这不是要求每一项都做。最小系统可以只有 CPU、Boot ROM、SRAM、UART、Timer 和简单地址译码。模块数量增多后，再引入标准总线、总线桥、DMA、缓存和外部存储器控制器。

### 3.1 各部分的职责边界

| 模块 | 它应该负责什么 | 不应偷偷承担什么 |
| --- | --- | --- |
| CPU 核 | 执行指令、发出访存请求、处理异常和中断 | 不能把某块板子的 UART 地址硬编码在流水线里 |
| CPU wrapper | 极性、端口、协议和参数适配 | 不宜塞入大量外设逻辑 |
| Interconnect | 地址译码、仲裁、请求和响应路由 | 不负责解释 GPIO 寄存器含义 |
| Memory wrapper | 把总线访问适配到 BRAM/SRAM/ROM IP | 不改变软件可见地址 |
| Peripheral | 实现寄存器、外部功能和中断源 | 不自行决定整个 SoC 的地址 |
| Interrupt controller | 汇集、屏蔽、记录、仲裁中断 | 不替外设保存全部业务状态 |
| `soc_top` | 实例化和连线 | 尽量少写复杂状态机 |
| `fpga_top` | 板级时钟、引脚和厂商 IP | 不放 CPU 架构相关逻辑 |

## 4. 片上互连的基本概念

### 4.1 主设备和从设备

在存储器映射的 SoC 中，主动发起访问的一方叫主设备或 Manager，被地址选中的一方叫从设备或 Subordinate。

常见主设备：

- CPU 指令取指口；
- CPU 数据访存口；
- DMA；
- 调试模块的系统总线访问口；
- 显示、网络等能主动搬运数据的控制器。

常见从设备：

- Boot ROM；
- SRAM；
- DDR 控制器；
- UART、GPIO、Timer；
- 系统控制和中断控制器；
- 未映射地址的默认错误从设备。

“UART 是外设”不代表它永远只能是从设备。普通 UART 寄存器接口是从设备；如果某个高速 UART 带 DMA，它的 DMA 通道还能成为总线主设备。

### 4.2 一次访问至少包含什么

不管采用自定义总线还是 AMBA，总要能表达以下信息：

```text
请求方要访问哪个地址
本次是读还是写
写入什么数据
哪些字节有效
从设备是否接受请求
读数据何时有效
访问是否成功
```

一个适合教学 SoC 的简化请求/响应接口可以写成：

```systemverilog
// request channel
logic        req_valid;
logic        req_ready;
logic [31:0] req_addr;
logic        req_write;
logic [31:0] req_wdata;
logic [3:0]  req_wstrb;

// response channel
logic        rsp_valid;
logic        rsp_ready;
logic [31:0] rsp_rdata;
logic        rsp_error;
```

这里已经包含了一个可扩展总线最重要的语义：

- `valid && ready` 时请求被接收；
- 请求被接收后，响应可以晚若干拍返回；
- `wstrb[3:0]` 分别控制 32 位数据的四个字节；
- `rsp_error` 让非法地址和设备错误不再表现为“读到一个不明所以的 0”。

如果 CPU 接口固定假设“读请求发出后一拍必有数据”，那么所有存储器和外设都必须满足这个假设，或者由 wrapper 插入等待。DDR、串口寄存器桥和跨时钟域访问都很难保证固定一拍，因此通用接口通常会带握手或等待信号。

### 4.3 地址译码与返回路由

地址译码器根据地址高位选中目标从设备：

```text
0x0000_1234 → Boot ROM
0x1000_0040 → SRAM
0x4000_1008 → UART
0x4000_2004 → GPIO
```

请求发出后，互连还要记住“这次访问去了哪里”，这样响应回来时才能送回原来的主设备。对于多周期访问，不能仅用“当前地址”选择返回数据，因为 CPU 可能已经准备下一次访问。

多主设备系统还需要仲裁。例如 CPU 和 DMA 同时请求 SRAM 时，仲裁器决定谁先访问，并保证另一方看到等待。更复杂的 AXI 系统可以允许多个未完成事务，互连还需利用事务 ID 配对响应。

## 5. 一份示例地址空间

下面只是便于理解的规划，不是 RISC-V、ARM 或 AMBA 强制规定的地址：

| 地址范围 | 大小 | 设备 | 属性 |
| --- | ---: | --- | --- |
| `0x0000_0000`–`0x0000_FFFF` | 64 KiB | Boot ROM / I-ROM | 只读、可执行 |
| `0x1000_0000`–`0x100F_FFFF` | 1 MiB | On-chip SRAM | 可读写、可执行 |
| `0x2000_0000`–`0x2000_FFFF` | 64 KiB | Local Timer / Software IRQ | MMIO、不可缓存 |
| `0x3000_0000`–`0x300F_FFFF` | 1 MiB | Interrupt Controller | MMIO、不可缓存 |
| `0x4000_0000`–`0x4000_0FFF` | 4 KiB | System Control | MMIO、不可缓存 |
| `0x4000_1000`–`0x4000_1FFF` | 4 KiB | UART0 | MMIO、不可缓存 |
| `0x4000_2000`–`0x4000_2FFF` | 4 KiB | GPIO0 | MMIO、不可缓存 |
| `0x4000_3000`–`0x4000_3FFF` | 4 KiB | Timer0 | MMIO、不可缓存 |
| `0x4001_0000`–`0x400F_FFFF` | 960 KiB | 外设扩展保留 | 保留 |
| `0x8000_0000`–`0xBFFF_FFFF` | 1 GiB | 外部 DDR / 大容量 RAM | 可缓存、可执行 |
| 其他地址 | — | Default Error Slave | 返回总线错误 |

规划地址空间时要检查：

- 范围不能重叠；
- 基地址最好按区域大小自然对齐；
- 每个外设预留固定窗口，例如 4 KiB；
- 给后续外设留扩展区；
- 明确哪些区域可执行、可缓存、只读或属于 MMIO；
- 链接脚本、BSP 头文件和 RTL 地址译码使用同一份定义。

真实项目最好维护一份机器可读的地址清单，例如 YAML/JSON，再生成 RTL package、C 头文件和文档，减少三处手工修改造成的不一致。

## 6. I-ROM、Boot ROM 和指令存储器

### 6.1 “I-ROM”可能指什么

教学项目里的 I-ROM 经常表示“CPU 的指令存储器”，但它未必真是流片中的掩膜 ROM。它可以由以下资源实现：

- FPGA BRAM，下载 bitstream 时预装程序；
- 片上 ROM；
- 片上 SRAM，先由 Boot ROM 搬入程序；
- QSPI Flash 的 XIP 窗口；
- 外部 DDR 中的一段可执行区域；
- 仿真用的行为模型和 `$readmemh` 文件。

所以设计文档应写“软件看到的行为”和“硬件实现方式”两项。软件可能都把它当作位于 `0x0000_0000` 的只读可执行空间，FPGA 版本和 ASIC 版本的底层实现却不同。

### 6.2 I-ROM 接口要说明的内容

至少确定：

| 项目 | 示例 |
| --- | --- |
| 地址宽度 | 16 位字节地址，对应 64 KiB |
| 数据宽度 | 32 位 |
| 地址单位 | 字节地址，不是 32 位字地址 |
| 读延迟 | 请求接受后 1 拍返回 |
| 是否握手 | `req/ready` 与 `rvalid` |
| 未对齐行为 | 取指地址要求 4 字节对齐，压缩指令时可能为 2 字节 |
| 是否支持写 | Boot ROM 不支持；仿真加载不属于 CPU 写接口 |
| 复位初始化 | 来自 `.mem`、`.hex` 或 FPGA IP 初始化文件 |

FPGA BRAM 通常是同步读：地址在时钟沿被采样，数据随后输出。若 CPU 以前连接的是组合读数组，换成 BRAM IP 后必须增加等待或流水寄存器，不能只把模块名替换掉。

### 6.3 Harvard 和统一存储结构

常见有两种连接方法。

**Harvard 形式**

```text
CPU I-Bus ─────▶ I-ROM / I-Cache / 指令互连
CPU D-Bus ─────▶ SRAM / MMIO / 数据互连
```

优点是取指和数据访问可以并行，接口直观。需要执行 SRAM 中的程序时，I-Bus 也必须能到达 SRAM；需要 CPU 写入指令区时，还要考虑缓存一致性或取指刷新。

**统一总线形式**

```text
CPU 统一主口 ──▶ Interconnect ──▶ ROM、SRAM、MMIO
```

互连更统一，但取指与数据访问会争用带宽。很多 CPU 对外仍给出独立指令口和数据口，进入 SoC 后再由仲裁器合并。

## 7. D-RAM、SRAM 与外部 DRAM

### 7.1 先消除一个名称歧义

不少教学项目把 `D-RAM` 写作 Data RAM，意思是“数据存储器”。它不一定是动态存储器 DRAM。FPGA 上最常见的数据存储器其实是 BRAM 或分布式 RAM。

如果文档中写“DRAM”，最好明确属于下面哪一种：

```text
Data RAM：按用途命名，存放 .data、.bss、heap、stack
Dynamic RAM：按器件技术命名，例如 DDR3/DDR4/LPDDR
```

前者可以直接做成片上 SRAM/BRAM；后者需要复杂的 DDR PHY 和控制器，通常使用 FPGA 厂商 IP。

### 7.2 数据 RAM 需要支持什么

CPU 的普通 load/store 至少需要：

- 32 位或 64 位读写数据；
- 字节写使能；
- 对齐检查或非对齐访问处理；
- 请求等待与读响应；
- 读写同址时的确定行为；
- 初始化策略；
- 访问越界错误。

32 位总线上的字节写使能例子：

| `wstrb` | 有效字节 | 常见指令 |
| --- | --- | --- |
| `4'b0001` | `wdata[7:0]` | `SB` 写最低字节 |
| `4'b0011` | `wdata[15:0]` | `SH` 写低半字 |
| `4'b1111` | 全部 32 位 | `SW` |

若 CPU 支持任意字节地址，写选通必须根据地址低位移动。`SH` 写到地址低两位为 `2'b10` 时，应形成 `4'b1100`，并把数据放到高半字节通道。

### 7.3 单口、简单双口和真双口

| 存储器形式 | 典型用途 | 需要处理的问题 |
| --- | --- | --- |
| 单口 RAM | 只有 CPU 数据口 | 同一拍只能完成一次访问 |
| 简单双口 RAM | 一口读、一口写 | 端口功能固定 |
| 真双口 RAM | CPU 与 DMA、或 I-Bus 与 D-Bus共享 | 同址冲突、写优先级 |

CPU 的 I-Bus 和 D-Bus 都要访问同一块 SRAM 时，可以使用双口 BRAM；也可以在前面放仲裁器共享单口 RAM。双口资源换取并行带宽，仲裁器则更节省存储器端口。

### 7.4 FPGA 中怎样接存储器 IP

建议在厂商 IP 外再包一层稳定的 RTL 接口：

```text
CPU/总线
   │ 通用 req/rsp、AHB 或 AXI
   ▼
memory_wrapper
   │ 厂商特定端口
   ▼
Xilinx Block Memory Generator / Intel on-chip memory / DDR controller
```

`memory_wrapper` 负责：

- 字节地址转成 BRAM 的字地址；
- 将 `wstrb` 转成 IP 的 byte enable；
- 吸收 IP 的固定读延迟；
- 生成 `ready/rvalid/error`；
- 在仿真时换成行为模型；
- 在综合时实例化厂商 IP。

仿真模型与 FPGA IP 不要求源代码相同，但从总线一侧观察到的时序必须一致。若仿真 RAM 是零延迟，FPGA BRAM 是一拍延迟，仿真通过并不能说明板上能工作。

## 8. CPU 核应该提供什么接口

一个可被 SoC 正常集成的 CPU 核，通常要暴露以下几组接口：

```text
时钟和复位
取指接口
数据访问接口
中断接口
调试接口（可选）
运行状态或性能计数接口（可选）
```

### 8.1 自定义 Harvard 接口示例

下面只描述语义，不要求照抄信号名：

```systemverilog
// Instruction port
output logic        imem_req_o;
output logic [31:0] imem_addr_o;
input  logic        imem_ready_i;
input  logic        imem_rvalid_i;
input  logic [31:0] imem_rdata_i;
input  logic        imem_error_i;

// Data port
output logic        dmem_req_o;
output logic [31:0] dmem_addr_o;
output logic        dmem_write_o;
output logic [31:0] dmem_wdata_o;
output logic [3:0]  dmem_wstrb_o;
input  logic        dmem_ready_i;
input  logic        dmem_rvalid_i;
input  logic [31:0] dmem_rdata_i;
input  logic        dmem_error_i;

// Interrupts
input  logic        timer_irq_i;
input  logic        software_irq_i;
input  logic        external_irq_i;
```

CPU 内部要保证：

- 请求没有被 `ready` 接受时，地址、写数据和控制信息保持稳定；
- 等待读响应时不会把另一笔响应错当成当前 load 的数据；
- 存储访问停顿能反压流水线或对应队列；
- `imem_error`、`dmem_error` 能形成访问异常；
- 中断进入、保存现场和 `mret`/异常返回符合指令集架构；
- 复位 PC 等于系统约定的 reset vector。

### 8.2 使用标准总线主口

另一种做法是让 CPU 直接给出 AXI、AHB-Lite 或 TileLink 等标准接口。SoC 互连和已有 IP 更容易复用，但 CPU 侧协议状态机也更复杂。

常见折中是：

```text
CPU 内部使用简单 req/rsp
           │
           ▼
cpu_bus_adapter
           │
           ▼
AXI4-Lite / AHB-Lite 系统总线
```

CPU 核只处理流水线和异常，总线 adapter 单独验证。以后更换总线协议时，CPU 主体不必大改。

## 9. 外设怎样接到 CPU

### 9.1 MMIO：把寄存器放进地址空间

Memory-Mapped I/O 的做法是让外设寄存器占用一段普通地址。CPU 使用 load/store 访问它们：

```c
#define GPIO_BASE 0x40002000u
#define GPIO_OUT  (*(volatile unsigned int *)(GPIO_BASE + 0x00u))

GPIO_OUT = 0x1u;
```

硬件路径是：

```text
CPU 执行 store
  → 数据总线发出地址 0x4000_2000、写数据 1
  → Interconnect 译码到 GPIO
  → GPIO 的 OUT 寄存器更新
  → gpio_o[0] 改变
  → fpga_top 的 I/O Buffer
  → FPGA 引脚/LED 电平改变
```

其中 `volatile` 告诉编译器，这个地址对应的读写具有外部副作用，不能随意删去或缓存到普通变量中。带数据 Cache 的系统还必须在页表、PMA/PMP 或缓存属性中把 MMIO 区标成不可缓存、设备类型和必要的强顺序属性。

### 9.2 GPIO 的常见寄存器

一个较完整的 GPIO 外设可以提供：

| 偏移 | 寄存器 | 访问 | 含义 |
| ---: | --- | --- | --- |
| `0x00` | `DATA_IN` | RO | 引脚输入值 |
| `0x04` | `DATA_OUT` | R/W | 输出值 |
| `0x08` | `OUTPUT_EN` | R/W | 输出使能 |
| `0x0C` | `OUT_SET` | WO | 写 1 置位相应输出位 |
| `0x10` | `OUT_CLEAR` | WO | 写 1 清零相应输出位 |
| `0x14` | `IRQ_ENABLE` | R/W | 中断使能 |
| `0x18` | `IRQ_TYPE` | R/W | 边沿/电平选择 |
| `0x1C` | `IRQ_POLARITY` | R/W | 上升/下降或高/低选择 |
| `0x20` | `IRQ_PENDING` | R/W1C | 中断挂起，写 1 清除 |

`OUT_SET` 和 `OUT_CLEAR` 能避免软件执行“读—改—写”时被中断打断。`W1C` 是 Write One to Clear，软件向某一位写 1 才清除该位。

外部 GPIO 输入通常与系统时钟异步。进入寄存器和边沿检测逻辑之前要做同步；机械按键还要防抖。多位异步数据若要求同一时刻一致，不能只给每一位各加两个触发器，需要握手、锁存或异步 FIFO。

### 9.3 UART、Timer 和其他外设

UART 通常分成三个部分：

```text
总线寄存器接口
├─ TXDATA / RXDATA
├─ STATUS / CONTROL
└─ BAUDDIV / IRQ

发送接收状态机
├─ 波特率计数
├─ 串并转换
└─ 起始位、数据位、校验位、停止位

FIFO（可选）
├─ TX FIFO
└─ RX FIFO
```

Timer 至少需要一个自由运行计数器和比较寄存器：

```text
counter >= compare 且 irq_enable = 1
                   ↓
               timer_irq = 1
```

SPI、I2C、PWM、Watchdog 也遵循同一连接形式：一侧是总线从设备寄存器，另一侧是具体外部协议或控制功能，中断线送往中断控制器。

## 10. 中断怎样从外设到达 CPU

中断路径可分成四段：

```text
事件发生
  → 外设形成 pending 状态
  → 中断控制器进行 enable、priority、route
  → CPU 收到 irq 输入并进入 Trap/Exception
  → 软件读取原因、服务设备、清除 pending
```

外设中断至少要定义：

- 电平触发还是边沿触发；
- pending 在什么时刻置位；
- 未使能时是否仍记录 pending；
- 软件怎样清除：读清、写 1 清、写 0 清或设备条件消失自动清；
- 多个事件同时出现时怎样保存；
- 中断正在服务期间又来一次同类事件怎样处理。

一个常见划分是：

| 中断类型 | 来源 | 连接方法 |
| --- | --- | --- |
| 本地定时器中断 | CPU 附近的 machine/system timer | 独立接入 CPU |
| 软件中断 | 核间通知或软件触发寄存器 | 独立接入 CPU |
| 外部中断 | UART、GPIO、SPI、DMA 等 | 汇入 PLIC、NVIC 或自定义控制器 |
| 不可屏蔽中断 | 看门狗、严重错误 | 独立高优先级入口，视 CPU 支持情况 |

RISC-V 项目常见 CLINT/ACLINT 类本地中断源和 PLIC/APLIC 类外部中断控制器；Cortex-M 常见 NVIC。名称不同，仍要回答“来源、屏蔽、优先级、挂起、认领和清除”这些问题。

## 11. AMBA 到底是什么

AMBA 是 Arm 提出的片上互连协议族，不是一根固定名称的“总线”。常见成员包括 AXI、AHB 和 APB。它们规定模块之间怎样交换地址、数据、响应和控制信息，使 CPU、存储器控制器和外设 IP 可以按统一契约连接。

使用 RISC-V CPU 并不妨碍采用 AMBA。CPU 指令集决定软件能执行哪些指令；AMBA 决定片上硬件模块怎样通信，两者位于不同层次。同样，RISC-V SoC 也可以使用 TileLink、Wishbone、AXI、AHB 或自定义总线。

### 11.1 AXI、AHB-Lite、APB 的差别

| 协议 | 适合的位置 | 主要特点 | 实现代价 |
| --- | --- | --- | --- |
| AXI4 | CPU、Cache、DMA、DDR、高带宽加速器 | 读写分离、五个通道、Burst、多笔未完成事务、ID | 最高 |
| AXI4-Lite | 控制寄存器、低吞吐 MMIO | AXI 的简化子集，无 Burst，通常单拍数据 | 中等，仍有五个独立通道 |
| AHB-Lite | 单主或较简单 MCU 系统总线、片上 SRAM | 地址/控制与数据阶段流水，接口比 AXI 简洁 | 中等偏低 |
| APB | UART、GPIO、Timer、系统控制 | Setup/Access 两阶段，无 Burst，吞吐要求低 | 最低 |

AXI 的写地址和写数据属于独立通道，不能假定它们总在同一拍到达。手写 AXI 从设备时，这是常见错误。AXI4-Lite 虽然去掉 Burst 和 ID 等复杂功能，仍需分别正确处理写地址、写数据和写响应。

AHB-Lite 采用流水化的地址阶段和数据阶段。一次传输的数据阶段与下一次传输的地址阶段可能重叠，因此从设备必须区分“当前数据属于上一拍的哪个地址”。

APB 的访问通常经历：

```text
IDLE
  → SETUP：PSEL=1，地址和控制有效
  → ACCESS：PENABLE=1，等待 PREADY
  → 完成或进入下一次 SETUP
```

它非常适合寄存器型外设，但不适合 DDR 或高吞吐 DMA 数据通路。

### 11.2 典型 AMBA 连接方式

常见 MCU SoC 使用分层总线：

```text
CPU / DMA
   │ AHB-Lite 或 AXI
   ▼
System Interconnect
   ├──────────────▶ Boot ROM
   ├──────────────▶ On-chip SRAM
   ├──────────────▶ DDR Controller
   └──────────────▶ AHB-to-APB / AXI-to-APB Bridge
                           │ APB
                           ├────▶ UART
                           ├────▶ GPIO
                           ├────▶ Timer
                           ├────▶ SPI / I2C
                           └────▶ System Control
```

桥接器把系统总线的一次访问转换为 APB 的 Setup/Access 时序。CPU 只知道自己在读写某个地址，不需要知道 UART 位于桥后。

### 11.3 是否一定要用 AMBA

不一定。可以按项目规模选择：

| 项目规模 | 可行选择 | 说明 |
| --- | --- | --- |
| 最小教学核 | 自定义 `req/rsp` | 信号少，容易看懂，必须把等待、错误和字节写清楚 |
| 单核 MCU | AHB-Lite + APB | 常见、复杂度适中、外设结构清楚 |
| FPGA 原型且大量使用厂商 IP | AXI4/AXI4-Lite + APB | 易连接 DDR、DMA、Vivado IP |
| 高性能多主系统 | AXI4/ACE/CHI 或其他 NoC | 需要 QoS、一致性和大量未完成事务 |

总线越标准，已有 IP 越容易连接；协议验证成本也会上升。对第一次完成 RTOS 移植的单核 RV32 项目，下面两种方案都合理：

```text
方案 A：CPU 简单 Harvard req/rsp
        + 小型自定义互连
        + ROM、SRAM、UART、GPIO、Timer

方案 B：CPU 简单接口
        + cpu_to_ahb_adapter
        + AHB-Lite 系统总线
        + AHB-to-APB bridge
        + APB 外设
```

若比赛要求展示规范的 SoC 集成能力，方案 B 的结构更接近常见 MCU；若时间紧，方案 A 更容易先把 CPU、操作系统和中断跑通。不要为了“看起来专业”直接从完整 AXI4 多主系统开始。

## 12. 三条典型数据路径

### 12.1 复位后的第一条指令

```text
外部复位有效
  → reset controller 让 CPU、总线、外设保持复位
  → 时钟稳定、PLL locked
  → 各时钟域同步释放复位
  → CPU 把 PC 设为 RESET_VECTOR
  → I-Bus 发出取指地址
  → Interconnect 选中 Boot ROM
  → ROM 返回指令
  → CPU 开始执行启动代码
```

只要 CPU 的 reset vector、RTL 地址译码、ROM 初始化地址和链接脚本起始地址有一项不一致，系统就无法启动。

### 12.2 软件点亮 GPIO

```text
程序执行 store 0x1 → GPIO_OUT
  → CPU D-Bus 请求
  → 系统互连地址译码
  → 总线桥（如果 GPIO 位于 APB）
  → GPIO 寄存器写入
  → gpio_o 改变
  → fpga_top 三态缓冲和引脚约束
  → LED 亮
```

测试这条路径时，既要看总线写地址和写数据，也要看寄存器是否真的更新以及顶层引脚有没有连接到正确端口。

### 12.3 Timer 触发 RTOS Tick

```text
Timer counter 到达 compare
  → Timer pending 置位
  → irq_enable 与 pending 形成中断请求
  → 中断控制器转发给 CPU
  → CPU 保存异常 PC 和原因，跳到 Trap 入口
  → RTOS Tick ISR 更新系统节拍并请求调度
  → 软件清除/重装 Timer
  → 上下文恢复，执行 mret/异常返回
```

这条路径跨越 Timer RTL、中断控制器、CPU CSR/Trap、汇编入口和 RTOS 内核。排错时应逐段确认信号，而不是只看“系统没有切换线程”。

## 13. 启动方式怎样影响 SoC

常见启动方式有三种。

### 13.1 FPGA BRAM 预初始化

```text
ELF
 → objcopy 生成 bin/hex
 → 转换成 .mem/.coe
 → 写入 BRAM 初始化文件
 → 重新生成 bitstream
 → FPGA 配置后 CPU 直接执行
```

实现最简单，修改程序通常要重新生成 bitstream。适合早期仿真和比赛演示。

### 13.2 Boot ROM 搬运

```text
CPU 从小型 Boot ROM 启动
 → 初始化 UART/SPI/DDR
 → 从 Flash 或串口读取主程序
 → 搬到 SRAM/DDR
 → 校验
 → 跳转到入口地址
```

这样可以只更新软件镜像，不重做整个 FPGA bitstream。Boot ROM 本身仍需要稳定的复位入口和最小驱动。

### 13.3 XIP

CPU 直接从 QSPI Flash 的映射窗口取指，叫 Execute In Place。它节省 SRAM，但 Flash 延迟较高，通常需要预取或 Cache；写 Flash 期间也要处理取指冲突。

## 14. 时钟、复位和跨时钟域

### 14.1 时钟规划

最小 SoC 最好先使用一个系统时钟。UART 波特率、Timer Tick 和 PWM 用时钟使能脉冲实现，不急于为每个外设生成新时钟域。

出现以下需求时才考虑多个时钟域：

- DDR 控制器要求专用用户时钟；
- 外设协议由独立参考时钟驱动；
- 高速 CPU 与低速 APB 分频；
- 功耗要求使用 clock gating；
- 异步外部接口需要恢复时钟。

每增加一个时钟域，都要增加 CDC 分析、异步 FIFO 或握手同步，以及该域独立的复位同步。

### 14.2 复位策略

常见策略是异步拉低、同步释放：

```text
外部复位或 PLL 未锁定
        │
        ▼
异步地让所有触发器尽快进入复位
        │
时钟恢复稳定
        ▼
每个时钟域经过两级或多级同步器释放复位
```

信号命名应明确极性，例如 `rst_ni` 表示低有效输入，`rst_o` 表示高有效输出。CPU wrapper 可以完成极性转换，但整套工程不要混用含义模糊的 `reset`、`rst`、`resetn`。

复位不是越多越好。大容量 RAM 数据阵列通常不需要逐位复位；软件通过启动代码清零 `.bss`。对所有 RAM 位增加复位，可能妨碍综合器推断 BRAM。

## 15. 推荐的通用工程目录

一套规模适中的 RTL/SoC 项目可按下面组织：

```text
project/
├─ docs/
│  ├─ soc_architecture.md
│  ├─ memory_map.md
│  ├─ interrupt_map.md
│  └─ register_map/
│
├─ rtl/
│  ├─ pkg/
│  │  ├─ soc_addr_pkg.sv
│  │  ├─ bus_pkg.sv
│  │  └─ irq_pkg.sv
│  ├─ top/
│  │  ├─ soc_top.sv
│  │  └─ fpga_top.sv
│  ├─ cpu/
│  │  ├─ cpu_core.sv
│  │  ├─ cpu_wrapper.sv
│  │  └─ cpu_bus_adapter.sv
│  ├─ bus/
│  │  ├─ interconnect.sv
│  │  ├─ address_decoder.sv
│  │  ├─ arbiter.sv
│  │  ├─ default_slave.sv
│  │  └─ ahb_to_apb_bridge.sv
│  ├─ memory/
│  │  ├─ boot_rom_wrapper.sv
│  │  ├─ sram_wrapper.sv
│  │  └─ sim_memory_model.sv
│  ├─ peripheral/
│  │  ├─ gpio/
│  │  ├─ uart/
│  │  ├─ timer/
│  │  └─ sysctrl/
│  ├─ interrupt/
│  ├─ clk_rst/
│  └─ debug/
│
├─ fpga/
│  ├─ ip/
│  ├─ constraints/
│  └─ scripts/
│
├─ tb/
│  ├─ unit/
│  ├─ soc/
│  ├─ models/
│  └─ assertions/
│
├─ sw/
│  ├─ boot/
│  ├─ bsp/
│  ├─ linker/
│  ├─ tests/
│  └─ applications/
│
├─ sim/
├─ scripts/
└─ build/                    生成物，不作为设计源文件
```

几个文件的长期职责应保持稳定：

| 文件 | 内容 |
| --- | --- |
| `soc_addr_pkg.sv` | RTL 使用的基地址和地址掩码 |
| `memory_map.md` | 人可读的内存与 MMIO 地址 |
| `soc.h` | 软件使用的外设基地址和寄存器定义 |
| `link.lds` | ROM/RAM 的软件段布局 |
| `interrupt_map.md` | 中断号、触发方式、清除方式 |
| `fpga_top.sv` | 板级引脚和 IP |
| `soc_top.sv` | 逻辑系统装配 |
| `filelist.f` | RTL 编译顺序和源文件清单 |

地址和中断号最好从单一来源生成。若暂时手工维护，每次修改都要同时核对 RTL、BSP、链接脚本和文档。

## 16. 三种规模的参考架构

### 16.1 最小教学型

```text
RV32 CPU
├─ I-Bus → Boot ROM
├─ D-Bus → 简单地址译码
│          ├─ SRAM
│          ├─ UART
│          ├─ GPIO
│          └─ Timer
└─ timer_irq + external_irq
```

适合先验证指令集、Trap 和 RTOS。总线可以是自定义 `valid/ready`，但应保留错误响应和字节写使能。

### 16.2 常见 MCU 型

```text
CPU I-Bus / D-Bus
       │
       ▼
AHB-Lite Interconnect
├─ Boot ROM
├─ SRAM
├─ Debug
└─ AHB-to-APB Bridge
   ├─ UART
   ├─ GPIO
   ├─ Timer
   ├─ SPI
   └─ System Control
```

这种结构适合单核、无 Cache 或小 Cache 的控制型 SoC。模块边界清楚，APB 外设容易复用。

### 16.3 高性能或较完整 FPGA SoC

```text
CPU + I/D Cache ── AXI ─┐
DMA ───────────── AXI ──┼─▶ AXI Interconnect / NoC
Debug System Bus ─ AXI ─┘       ├─ DDR Controller
                                ├─ On-chip SRAM
                                ├─ Accelerator
                                └─ AXI-to-APB Bridge
                                   └─ 低速外设
```

此时还要处理 Cache 一致性、DMA 可见性、Burst、多个 outstanding transaction、ID、QoS、跨时钟域和 DDR 校准。它适合已有可靠总线 IP 和验证环境的团队，不宜作为第一次 RTOS 移植的起点。

## 17. 验证顺序

通用 SoC 不应只靠“跑一个 C 程序看 UART 有没有输出”验证。建议按层推进。

### 17.1 模块级

- ROM：逐地址读回初始化内容，检查读延迟；
- SRAM：字、半字、字节写，检查 byte enable；
- GPIO：寄存器读写、输入同步、边沿中断、W1C；
- UART：波特率、发送帧、接收帧、FIFO 和中断；
- Timer：计数、比较、周期模式和清中断；
- 总线桥：等待、错误、背靠背传输；
- 地址译码：边界地址、未映射地址和重叠检查。

### 17.2 SoC 级

先使用很短的裸机程序：

1. 复位后写一个签名到仿真端口；
2. 测试 SRAM 读写；
3. 写 GPIO；
4. UART 发送字符；
5. Timer 产生一次中断；
6. 中断返回后继续执行；
7. 再运行 RTOS 的双线程切换；
8. 最后运行 CoreMark 或比赛应用。

### 17.3 必看的断言和波形

可加入以下协议断言：

- `valid && !ready` 时请求字段保持稳定；
- 每个已接受的读请求最终得到一次响应；
- 不会无请求地产生响应；
- 同一时刻只能选中合法数量的从设备；
- 未映射地址必须进入 default slave；
- APB `PENABLE` 只能出现在 Setup 之后；
- AXI 各通道分别遵守 `VALID/READY` 稳定规则；
- 复位释放后不存在 X 态传播到总线控制信号。

波形排查以请求链为主：

```text
CPU request
 → address decode
 → slave select
 → slave accept
 → slave response
 → interconnect response
 → CPU commit / trap
```

### 17.4 FPGA 验证

上板时逐步增加功能：

```text
时钟与复位 LED
 → 固定 GPIO 翻转
 → CPU 从 BRAM 启动
 → UART 轮询输出
 → Timer 中断
 → RTOS 线程
 → 外部存储器和完整应用
```

可使用 ILA/SignalTap 观察 PC、总线地址、请求、响应、中断和 Trap 原因。不要把 CPU 内部几百个信号全部接进逻辑分析仪，先围绕一条失败的数据路径选信号。

## 18. 常见失误

### 18.1 把零延迟仿真 RAM 直接换成同步 BRAM

结果通常是 CPU 取到上一地址的数据。应在接口中表达读延迟，或者用 wrapper 适配。

### 18.2 只译码请求，没有保存响应来源

多周期从设备返回时，“当前地址”已经改变。互连需要保存目标从设备编号，或使用带 ID 的协议。

### 18.3 省略字节写使能

`SB`、`SH`、设备寄存器局部字段写入会错误。CPU、总线、RAM wrapper 三处都要保持 byte enable 语义。

### 18.4 默认从设备永远返回 0

非法地址被掩盖，软件可能继续运行并在更远处出错。应产生总线错误，并让 CPU形成 load/store access fault；早期调试至少要记录错误地址。

### 18.5 MMIO 被 Cache 或乱序访问

GPIO、UART 等寄存器不能按普通缓存内存处理。高性能 CPU 还需要 fence/barrier 和正确的内存属性。

### 18.6 顶层同时混入板级和 SoC 逻辑

更换 FPGA 时会牵动 CPU 和外设；仿真也被迫实例化 PLL、DDR PHY 等原语。把 `fpga_top` 和 `soc_top` 分开。

### 18.7 中断只接了一根线，没有定义 pending 语义

脉冲可能在 CPU 屏蔽中断时丢失，电平中断可能因为没有清源而反复进入。每个中断都要写明触发、保持和清除方式。

### 18.8 UART 和 Timer 使用了错误的时钟频率

软件计算波特率和 Tick 周期时必须知道外设实际输入时钟。PLL 参数、RTL 常量和 BSP 的 `CPU_FREQ_HZ` 不能互相矛盾。

### 18.9 复位整个 RAM

大量复位逻辑可能让 BRAM 推断失败。RAM 内容初始化和 CPU/外设控制寄存器复位要分开考虑。

## 19. 对第一次完整 SoC/RTOS 项目的建议

如果目标是让一个自研 RV32 CPU 稳定运行 RT-Thread 一类 RTOS，建议先收敛到下面的硬件范围：

```text
必需：
  RV32 CPU，含 CSR、异常、中断、mret
  独立或可仲裁的指令/数据访问接口
  Boot ROM 或可初始化的指令存储器
  可读写 SRAM，支持字节写
  UART
  系统 Timer
  外部中断入口或小型中断控制器
  地址译码和默认错误响应
  稳定的时钟与复位

第二阶段：
  GPIO
  更完整的中断控制器
  调试模块
  性能计数器
  软件可更新的启动方式

以后再加：
  Cache
  DMA
  DDR
  多主 AXI
  多核和一致性
```

总线选择上，可以先保留 CPU 内部的简单 `req/rsp`，在 CPU 外增加 adapter。若团队希望体现常见 MCU 架构，可采用 AHB-Lite 作为系统总线、APB 作为低速外设总线；若项目主要依赖 Vivado 的 DDR 和 DMA IP，AXI4/AXI4-Lite 更方便。

在架构冻结前，团队至少应共同确认四张表：

1. 顶层端口表；
2. 地址空间表；
3. 中断映射表；
4. 时钟和复位表。

这四张表确定后，CPU、SoC、BSP 和应用才能并行开发。否则一个人修改 UART 地址，另一个人的链接脚本和驱动就会同时失效。

## 20. 设计完成检查表

### 顶层与板级

- [ ] `fpga_top` 的端口与引脚约束一致；
- [ ] `soc_top` 不依赖某块开发板的物理引脚命名；
- [ ] 双向引脚在 SoC 内部已拆成 `i/o/oe`；
- [ ] PLL lock 参与复位释放；
- [ ] 每个时钟域都有自己的复位同步。

### CPU 与总线

- [ ] 复位 PC 与 Boot ROM 地址一致；
- [ ] 请求等待时地址和控制保持稳定；
- [ ] 读响应与原请求正确配对；
- [ ] 支持字节写使能；
- [ ] 非法访问能报错并形成异常；
- [ ] 指令访问和数据访问冲突已有明确仲裁规则。

### 存储器

- [ ] I-ROM/SRAM 的地址单位统一为字节地址；
- [ ] 仿真模型与 FPGA IP 的读延迟一致；
- [ ] `.text/.data/.bss/heap/stack` 都落在合法区域；
- [ ] RAM 深度、地址宽度和链接脚本长度一致；
- [ ] 双口同址冲突行为已定义。

### 外设与中断

- [ ] 每个外设具有唯一且对齐的地址窗口；
- [ ] 寄存器访问类型、复位值和副作用明确；
- [ ] MMIO 区域不可缓存；
- [ ] 每个中断的触发、pending、enable 和清除方式明确；
- [ ] Timer 输入时钟与 BSP 计算一致；
- [ ] UART 波特率在仿真和 FPGA 上都验证过。

### 验证和交付

- [ ] 地址边界与未映射地址已有测试；
- [ ] 裸机启动、SRAM、UART、Timer 中断依次通过；
- [ ] RTOS Tick 和上下文切换通过；
- [ ] FPGA 上能重复启动并稳定输出；
- [ ] RTL、BSP、链接脚本和文档使用同一份地址/中断定义；
- [ ] 生成文件与手写源文件分目录保存。

## 21. 官方资料

- [Arm：Introduction to AMBA AXI4](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf)  
  适合先理解 AXI 的五个通道、VALID/READY 握手、读写事务和 Burst。
- [Arm：AMBA AXI and ACE Protocol Specification](https://developer.arm.com/documentation/ihi0022/latest/)  
  实现或审核 AXI 接口时使用的正式规范。
- [Arm：AMBA 3 AHB-Lite Protocol Specification](https://developer.arm.com/documentation/ihi0033/latest/)  
  适合 MCU 系统总线和较简单的内存映射互连。
- [Arm：AMBA APB Protocol Specification](https://developer.arm.com/documentation/ihi0024/latest/)  
  适合 UART、GPIO、Timer 等低带宽寄存器型外设。

阅读规范时不需要一开始记住全部信号。先画清本项目的主设备、从设备、地址空间和响应延迟，再对照所选协议检查每条通道的时序。协议名本身不会让 SoC 自动正确，真正决定可用性的仍是接口契约、异常处理和逐层验证。
