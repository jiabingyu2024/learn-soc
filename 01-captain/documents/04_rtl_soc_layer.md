# RTL/SoC 层：CPU、存储器、外设与 FPGA 顶层

> 上层接口：[BSP 与软件移植层](./03_bsp_porting_layer.md)  
> 操作系统：[RT-Thread 层](./02_rt_thread_layer.md)  
> 应用：[应用层](./01_application_layer.md)  
> 通用架构补充：[从顶层引脚到片上总线](./05_generic_rtl_soc_architecture.md)

## 1. RTL/SoC 层最终要提供什么

RTL/SoC 层要给软件提供一个稳定的“机器”：

```text
复位后从固定地址取指
IROM 中的指令可执行
DRAM 中的数据可读写
MMIO 地址可访问 UART、Timer、LED 等
Timer 能产生机器定时器中断
UART 能产生外部中断
CPU 能正确进入和退出 Trap
cycle/instret 可用于性能统计
```

软件只依赖三类硬件接口：

1. 地址：IROM、DRAM 和 MMIO；
2. 中断：Timer、External，以及对应 CSR cause；
3. 时序：存储器请求何时接受、读数据何时有效。

这三类接口确定后，不应在移植过程中频繁变化。

## 2. 建议把“板级顶层”和“SoC 顶层”分开

当前工程已经有：

```text
top.sv
└─ student_top.sv
   ├─ myCPU
   ├─ IROM_0
   └─ SocMemBridge
```

建议继续保留两层：

### 2.1 `top.sv`：FPGA 板级顶层

只处理：

- FPGA 差分时钟引脚；
- PLL；
- PLL lock 到复位的转换；
- 物理 UART 引脚归属；
- 板级 XDC 端口；
- 必要的时钟域跨越；
- 实例化 SoC 顶层。

### 2.2 `student_top.sv` 或 `soc_top.sv`：纯 SoC 顶层

处理：

- CPU；
- IROM；
- DRAM；
- 地址译码；
- MMIO 外设；
- Timer IRQ；
- External IRQ；
- 软件可见 LED/SEG/SW/KEY；
- 仿真调试端口。

这样 Verilator 可以直接实例化 SoC 顶层，不必模拟 FPGA 差分时钟和 PLL。

## 3. 最终板级顶层接口

基于当前工程和比赛 digital-twin 端口，板级顶层可保持：

```systemverilog
module top (
    input  wire        i_sys_clk_p,
    input  wire        i_sys_clk_n,
    input  wire        i_uart_rx,
    output wire        o_uart_tx,

    output wire [31:0] virtual_led,
    output wire [39:0] virtual_seg
);
```

如果开发板确实有独立复位引脚，可增加：

```systemverilog
input wire i_reset_n;
```

如果没有，则继续使用：

```text
PLL 未 lock → SoC 保持复位
PLL lock 后 → 每个时钟域同步释放复位
```

### 3.1 板级 UART 的归属

当前物理 UART 接给 `twin_controller`，CPU 无法把它当 RT-Thread console 使用。最终必须选择一种方案：

#### 方案 A：比赛 OS 模式直接使用 console

```text
i_uart_rx/o_uart_tx
       ↓
student_top 内的 uart_mmio
       ↓
CPU MMIO
```

virtual switch/key 可暂时置零，LED/SEG 仍由 CPU MMIO 控制。

#### 方案 B：保留 digital-twin，另用第二路 UART

前提是开发板有第二路可用串口引脚。

#### 方案 C：扩展 digital-twin 协议

把 console 字节作为一种报文在同一 UART 上传输。需要同时修改 FPGA 和上位机协议，工作量较大。

第一版 RT-Thread 推荐方案 A。不要让两个发送模块同时驱动 `o_uart_tx`。

## 4. 最终 SoC 顶层接口

建议把 SoC 顶层整理为：

```systemverilog
module student_top #(
    parameter int unsigned P_SW_CNT          = 64,
    parameter int unsigned P_LED_CNT         = 32,
    parameter int unsigned P_SEG_CNT         = 40,
    parameter int unsigned P_KEY_CNT         = 8,

    parameter logic [31:0] P_IROM_ADDR_START = 32'h8000_0000,
    parameter logic [31:0] P_IROM_ADDR_END   = 32'h8004_0000,
    parameter logic [31:0] P_DRAM_ADDR_START = 32'h8010_0000,
    parameter logic [31:0] P_DRAM_ADDR_END   = 32'h8014_0000
) (
    input  logic                      w_cpu_clk,
    input  logic                      w_clk_50Mhz,
    input  logic                      w_clk_rst,

    input  logic                      uart_rx,
    output logic                      uart_tx,

    input  logic [P_KEY_CNT - 1:0]    virtual_key,
    input  logic [P_SW_CNT  - 1:0]    virtual_sw,
    output logic [P_LED_CNT - 1:0]    virtual_led,
    output logic [P_SEG_CNT - 1:0]    virtual_seg

`ifdef VERILATOR_TB
    ,
    output logic [31:0]               dbg_perip_addr,
    output logic [31:0]               dbg_perip_wdata,
    output logic [3:0]                dbg_perip_mask,
    output logic                      dbg_perip_wen,

    output logic [63:0]               dbg_perf_cycle,
    output logic [63:0]               dbg_perf_commit,

    output logic                      dbg_commit_valid,
    output logic [31:0]               dbg_commit_pc,
    output logic [31:0]               dbg_commit_inst,
    output logic                      dbg_commit_is_trap,
    output logic [31:0]               dbg_commit_cause
`endif
);
```

`w_clk_rst` 当前是高有效复位。若以后改成低有效，应同步修改名称，例如 `rst_ni`，避免端口名和极性矛盾。

SoC 顶层不需要把 Timer IRQ 暴露到 FPGA 引脚。Timer、UART 和中断控制器都在 SoC 内部，IRQ 直接接 CPU。

## 5. 建议的 SoC 内部层次

```text
student_top
├─ reset_sync_cpu
├─ reset_sync_50m
├─ myCPU
│  └─ core_top
├─ IROM_0
├─ SocMemBridge
│  ├─ DramBramAdapter
│  │  └─ DRAM_0
│  ├─ legacy_gpio_mmio
│  ├─ uart_mmio
│  ├─ machine_timer
│  └─ irq_ctrl（第二阶段）
└─ debug/perf wiring
```

推荐让 `uart_mmio` 和 `machine_timer` 都工作在 CPU 时钟域。这样 CPU MMIO 不需要跨时钟域桥。

`display_seg` 可以继续运行在 50 MHz 域，CPU 写入值经过同步后显示。

## 6. CPU 核与 SoC 的接口

### 6.1 基于当前工程的最小改动接口

当前 `myCPU` 接口已经包含固定延迟 IROM 和 ready/valid D-MEM。建议保留并增加 IRQ：

```systemverilog
module myCPU (
    input  logic        cpu_rst,
    input  logic        cpu_clk,

    /* 固定一拍同步 IROM */
    output logic [31:0] irom_addr,
    input  logic [31:0] irom_data,
    output logic        irom_ena,

    /* 数据和 MMIO 请求 */
    output logic        dmem_req_valid,
    input  logic        dmem_req_ready,
    output logic        dmem_req_write,
    output logic [31:0] dmem_req_addr,
    output logic [31:0] dmem_req_wdata,
    output logic [3:0]  dmem_req_wstrb,
    output logic        dmem_req_uncached,

    input  logic        dmem_resp_valid,
    input  logic [31:0] dmem_resp_rdata,

    /* 机器模式中断 */
    input  logic        irq_software,
    input  logic        irq_timer,
    input  logic        irq_external
);
```

### 6.2 IROM 时序合同

继续使用当前固定一拍模型：

```text
T0 上升沿前：
    irom_ena=1
    irom_addr=A

T0 上升沿：
    IROM 锁存 A 的 word address

T0 上升沿后：
    irom_data 对应地址 A
    core 前端必须把该数据与请求时 PC 对齐
```

此接口没有 `irom_resp_valid`，所以只能用于固定延迟 IROM。若将来接 Cache、AXI 或可变延迟存储器，才值得把取指接口改成完整 ready/valid。

现阶段重新设计取指握手会影响前端和性能，不是 RT-Thread 的必需修改。

### 6.3 D-MEM 请求合同

请求有效：

```text
dmem_req_valid && dmem_req_ready
```

表示该周期请求被接受。

写请求：

```text
dmem_req_write = 1
dmem_req_wdata
dmem_req_wstrb
```

读请求：

```text
dmem_req_write = 0
```

读数据稍后通过：

```text
dmem_resp_valid
dmem_resp_rdata
```

返回。

当前接口没有写响应。SoC 必须保证写请求在 `req_ready` 接受后会产生且只产生一次副作用。

### 6.4 `dmem_req_uncached`

建议语义固定为：

```text
DRAM 普通地址：可缓存
所有 MMIO：强制 uncached
```

CPU 核负责：

- MMIO 不进入 DCache；
- 不合并 MMIO 写；
- 保持 uncached 访问顺序；
- `FENCE` 等待更早访问完成。

SoC 负责按地址选择外设，不依赖 `uncached` 替代地址译码。

## 7. 建议的地址映射

| 地址范围 | 模块 | 说明 |
|---|---|---|
| `0x8000_0000..0x8003_FFFF` | IROM | 256 KiB 指令 |
| `0x8010_0000..0x8013_FFFF` | DRAM | 256 KiB 数据 |
| `0x8020_0000` | SW0 | 现有 |
| `0x8020_0004` | SW1 | 现有 |
| `0x8020_0010` | KEY | 现有 |
| `0x8020_0020` | SEG | 现有 |
| `0x8020_0040` | LED | 现有 |
| `0x8020_0050` | performance counter | 现有 |
| `0x8020_1000..0x8020_10FF` | MMIO UART | 新增 |
| `0x8020_2000..0x8020_20FF` | machine timer | 新增 |
| `0x8020_3000..0x8020_30FF` | IRQ controller | 第二阶段 |

地址上界采用排他写法：

```text
base <= addr < end
```

IROM 地址越界不能仅截取低位后回绕。仿真中应对越界取指报警。

## 8. IROM 如何适配

### 8.1 容量

当前 IROM：

```text
4096 words × 32 bit = 16 KiB
irom_word_addr = irom_addr[13:2]
```

推荐：

```text
65536 words × 32 bit = 256 KiB
irom_word_addr = irom_addr[17:2]
```

RTL：

```systemverilog
logic [15:0] irom_word_addr;

assign irom_word_addr =
    (irom_addr - P_IROM_ADDR_START) [17:2];
```

SystemVerilog 对表达式切片的工具兼容性不完全一致，也可以先定义：

```systemverilog
logic [31:0] irom_offset;

assign irom_offset    = irom_addr - P_IROM_ADDR_START;
assign irom_word_addr = irom_offset[17:2];
```

### 8.2 Vivado Block Memory Generator 配置

建议：

```text
Memory Type                       Single Port ROM
Write Width A                     32
Write Depth A                     65536
Read Width A                      32
Enable A                          Use ENA Pin
Output register of memory         false
Output register of core           false
Load Init File                    true
COE                               firmware_imem.coe
```

Tcl 示例：

```tcl
create_ip -name blk_mem_gen \
          -vendor xilinx.com \
          -library ip \
          -version 8.4 \
          -module_name IROM_0

set_property -dict [list \
    CONFIG.Memory_Type {Single_Port_ROM} \
    CONFIG.Write_Width_A {32} \
    CONFIG.Write_Depth_A {65536} \
    CONFIG.Read_Width_A {32} \
    CONFIG.Enable_A {Use_ENA_Pin} \
    CONFIG.Register_PortA_Output_of_Memory_Primitives {false} \
    CONFIG.Register_PortA_Output_of_Memory_Core {false} \
    CONFIG.Load_Init_File {true} \
    CONFIG.Coe_File $irom_coe \
] [get_ips IROM_0]
```

### 8.3 Verilator 行为模型

参数同步改为：

```systemverilog
parameter int unsigned ADDR_WIDTH = 16;
```

仍保持：

```systemverilog
always_ff @(posedge clka)
    if (ena)
        addra_q <= addra;

assign douta = mem[addra_q];
```

不能让 Verilator 用组合零延迟 IROM，而 FPGA 用同步 BRAM。两者不同会导致 PC 和指令错位。

### 8.4 初始化

软件构建生成：

```text
firmware_imem.mem
firmware_imem.coe
```

其中第 0 个 word 对应：

```text
0x8000_0000
```

第 1 个 word 对应：

```text
0x8000_0004
```

RISC-V 是小端，但 COE 的每个 32 位 word 应按 CPU 看到的指令数值书写。例如内存字节：

```text
13 00 00 00
```

对应 COE word：

```text
00000013
```

### 8.5 IROM 中只放什么

当前数据口不能读取 IROM，因此第一版只放：

```text
.start
.text
```

不要放：

```text
.rodata
.data
FSymTab
VSymTab
.rti_fn*
CoreMark 数据
```

这些 section 需要由 load 指令读取，应初始化到 DRAM。

## 9. DRAM 如何适配

### 9.1 容量和地址

当前 DRAM 已是：

```text
65536 words × 32 bit = 256 KiB
0x8010_0000..0x8013_FFFF
```

容量可以先保持。

SoC 译码后传给适配器的地址应是：

```systemverilog
req_addr - 32'h8010_0000
```

适配器再取 word index：

```systemverilog
req_addr[17:2]
```

这里的 `req_addr` 已是相对 DRAM base 的 offset。

### 9.2 Vivado IP 配置

当前配置方向可继续使用：

```text
Memory Type                       Single Port RAM
Write Width A                     32
Write Depth A                     65536
Read Width A                      32
Byte Write Enable                 true
Byte Size                         8
Operating Mode                    READ_FIRST
Primitive Output Register         true
Core Output Register              false
REGCEA                            true
Load Init File                    true
```

Tcl：

```tcl
create_ip -name blk_mem_gen \
          -vendor xilinx.com \
          -library ip \
          -version 8.4 \
          -module_name DRAM_0

set_property -dict [list \
    CONFIG.Memory_Type {Single_Port_RAM} \
    CONFIG.Write_Width_A {32} \
    CONFIG.Write_Depth_A {65536} \
    CONFIG.Read_Width_A {32} \
    CONFIG.Enable_A {Use_ENA_Pin} \
    CONFIG.Use_Byte_Write_Enable {true} \
    CONFIG.Byte_Size {8} \
    CONFIG.Operating_Mode_A {READ_FIRST} \
    CONFIG.Register_PortA_Output_of_Memory_Primitives {true} \
    CONFIG.Register_PortA_Output_of_Memory_Core {false} \
    CONFIG.Use_REGCEA_Pin {true} \
    CONFIG.Load_Init_File {true} \
    CONFIG.Coe_File $dram_coe \
] [get_ips DRAM_0]
```

### 9.3 当前 `DramBramAdapter` 的职责

它负责：

- 总线字节地址转 BRAM word address；
- 根据 `addr[1:0]` 移动写数据；
- 移动 byte write enable；
- 保存 load 的 byte offset；
- 把 BRAM word 右移后返回；
- 产生两拍后的 `resp_valid`。

写对齐：

```systemverilog
dram_wdata = req_wdata << (req_addr[1:0] * 8);
dram_we    = req_wstrb << req_addr[1:0];
```

读对齐：

```systemverilog
resp_rdata = dram_rdata_raw >> (saved_offset * 8);
```

CPU 核再根据 LB/LBU/LH/LHU/LW 完成截取和符号扩展。

### 9.4 DRAM 时序合同

当前 IP 和适配器约定读返回为固定两拍：

```text
T0：req_valid/ready 接受读请求
T1：BRAM 内部读数据产生
T2：DOUT 与 resp_valid 对应
```

Verilator `DRAM_0.sv` 必须复制相同：

- READ_FIRST；
- byte write；
- primitive output register；
- REGCEA；
- 两拍 valid。

### 9.5 DMEM 初始化

软件工具生成：

```text
firmware_dmem.mem
firmware_dmem.coe
```

第 0 个 word 对应：

```text
0x8010_0000
```

应包含：

- `.rt_rodata`；
- `.data/.sdata`；
- 字符串；
- RT-Thread init table；
- FinSH 命令表；
- CoreMark 静态初值。

`.bss` 不需要写进文件，由 `start.S` 清零。

### 9.6 是否要改成双口 RAM

第一版单口 RAM 足够。

需要运行时下载或调试访问时，可改为 Simple Dual Port：

```text
Port A：CPU
Port B：Bootloader、JTAG loader 或调试器
```

双口会引入同地址冲突语义和更多验证，不应在 RT-Thread 首次启动前强行加入。

## 10. MMIO 总线和地址译码

### 10.1 `SocMemBridge` 的职责

```text
CPU dmem request
  ↓
地址译码
  ├─ DRAM
  ├─ GPIO/LED/SEG
  ├─ UART
  ├─ Timer
  └─ IRQ controller
  ↓
选中的 slave 接受请求
  ↓
读返回仲裁
  ↓
CPU dmem response
```

建议每个请求只能命中一个 slave。仿真中加入 one-hot 检查。

### 10.2 地址选择

```systemverilog
dram_sel  = addr >= DRAM_BASE  && addr < DRAM_END;
uart_sel  = addr >= UART_BASE  && addr < UART_END;
timer_sel = addr >= TIMER_BASE && addr < TIMER_END;
irq_sel   = addr >= IRQ_BASE   && addr < IRQ_END;
```

未知 MMIO 访问第一版可返回 0，但应：

- 记录调试告警；
- 保证 `resp_valid` 不会永久缺失；
- 后续可升级为 load/store access fault。

### 10.3 读返回

由于 DRAM 两拍、简单 MMIO 可一拍，需要保存请求来源：

```text
T0 接受读请求并记录 slave_id
T1/T2 等对应 slave 的 resp_valid
返回对应 rdata
```

不要只看当前总线地址选择返回数据。CPU 在响应周期可能已经发出另一个地址。

### 10.4 写请求

简单 MMIO 可在请求接受的上升沿更新寄存器。对于 UART：

- TX busy 时 `req_ready` 可拉低；
- 或总线始终接受，但内部必须有 TX FIFO；
- 不能接受后静默丢字节。

第一版最容易的方案：

```text
UART TX busy → uart_req_ready=0
软件轮询 STATUS 后再写 TXDATA
```

## 11. CPU 可见 UART

### 11.1 模块接口

```systemverilog
module uart_mmio #(
    parameter int unsigned CLK_FREQ_HZ = 50_000_000,
    parameter int unsigned BAUD_RATE   = 115_200
) (
    input  logic        clk,
    input  logic        rst,

    input  logic        req_valid,
    output logic        req_ready,
    input  logic        req_write,
    input  logic [7:0]  req_offset,
    input  logic [31:0] req_wdata,
    input  logic [3:0]  req_wstrb,
    output logic        resp_valid,
    output logic [31:0] resp_rdata,

    input  logic        uart_rx,
    output logic        uart_tx,
    output logic        irq
);
```

### 11.2 寄存器

| 偏移 | 名称 | 说明 |
|---:|---|---|
| `0x00` | TXDATA | 写低 8 位发送 |
| `0x04` | RXDATA | 读低 8 位；读取可清 valid |
| `0x08` | STATUS | TX busy、RX valid |
| `0x0C` | CTRL | RX IRQ enable 等 |
| `0x10` | IRQ_STATUS | pending 状态 |
| `0x14` | BAUD_DIV | 可选 |

### 11.3 接收同步

`uart_rx` 对 CPU 时钟异步，至少经过两级同步器再采样：

```systemverilog
(* ASYNC_REG = "TRUE" *) logic rx_meta;
(* ASYNC_REG = "TRUE" *) logic rx_sync;
```

UART RX 状态机、波特率采样和 IRQ 都在 CPU 时钟域完成。

### 11.4 第一版与第二版

```text
第一版：
TX polling
RX polling
1-byte holding register

第二版：
RX FIFO
RX interrupt
overrun 标志
可选 TX FIFO
```

FinSH 连续输入时，只有 1-byte holding register 容易丢字符，最终应至少有一个小型 RX FIFO。

## 12. Machine Timer

### 12.1 模块接口

```systemverilog
module machine_timer (
    input  logic        clk,
    input  logic        rst,

    input  logic        req_valid,
    output logic        req_ready,
    input  logic        req_write,
    input  logic [7:0]  req_offset,
    input  logic [31:0] req_wdata,
    input  logic [3:0]  req_wstrb,
    output logic        resp_valid,
    output logic [31:0] resp_rdata,

    output logic        timer_irq
);
```

### 12.2 内部寄存器

```systemverilog
logic [63:0] mtime;
logic [63:0] mtimecmp;
logic        enable;

assign timer_irq = enable && (mtime >= mtimecmp);
```

`mtime` 每个 CPU 时钟加 1。若 CPU 时钟可改变，BSP 必须使用一致的 `SOC_CPU_FREQ_HZ`。

### 12.3 寄存器地址

| 偏移 | 名称 |
|---:|---|
| `0x00` | MTIME_LO |
| `0x04` | MTIME_HI |
| `0x08` | MTIMECMP_LO |
| `0x0C` | MTIMECMP_HI |
| `0x10` | CTRL |

### 12.4 IRQ 行为

```text
mtime < mtimecmp  → timer_irq=0
mtime >= mtimecmp → timer_irq=1
```

软件把 `mtimecmp` 更新到未来后，IRQ 自动下降。`timer_irq` 接 CPU 的 `irq_timer`，并反映到 `mip.MTIP`。

## 13. 外部中断控制器

### 13.1 最小方案

只有 UART 时：

```text
uart_irq → CPU irq_external
```

进入 cause 11 后，BSP 直接读取 UART IRQ status。

### 13.2 多外设方案

新增 `irq_ctrl.sv`：

```text
source irq
  ↓
pending
  ↓ 与 enable
priority select
  ↓
irq_external
```

MMIO 至少包括：

| 寄存器 | 作用 |
|---|---|
| `PENDING` | 查看等待中断 |
| `ENABLE` | 单源使能 |
| `CLAIM` | 读出当前最高优先级 ID |
| `COMPLETE` | 写 ID 表示处理完成 |

如果只为比赛移植，不需要完整实现所有 PLIC 上下文和大规模优先级寄存器。

## 14. CPU 核需要增加什么

SoC 外设产生 IRQ 后，CPU 核必须正确接收。

### 14.1 CSR

至少：

```text
mstatus  0x300：MIE/MPIE/MPP
mie      0x304：MSIE/MTIE/MEIE
mtvec    0x305
mscratch 0x340
mepc     0x341
mcause   0x342
mtval    0x343
mip      0x344：MSIP/MTIP/MEIP
mhartid  0xF14：读 0
```

`mip` 的硬件位：

```systemverilog
mip[3]  = irq_software;
mip[7]  = irq_timer;
mip[11] = irq_external;
```

### 14.2 中断判定

```systemverilog
global_enable = mstatus_mie;
pending = mie & mip;
```

只有 `global_enable && pending != 0` 才能进入中断。

可先规定：

```text
External > Software > Timer
```

如果同时 pending，选择一个并保持其他 pending，退出后再处理。

### 14.3 `mcause`

当前 cause 接口只有 5 位，需要改为 32 位：

```text
异常：mcause[31]=0
中断：mcause[31]=1
```

```text
Machine software interrupt：0x80000003
Machine timer interrupt：   0x80000007
Machine external interrupt：0x8000000B
```

### 14.4 精确中断

中断只在架构提交边界接受：

```text
保留已提交指令
排空或妥善处理已提交但未完成的 store
清除所有年轻指令
mepc = 下一条尚未执行的架构 PC
mcause = 中断 cause
mtval = 0
MPIE = MIE
MIE = 0
PC = mtvec
```

当前顺序提交结构适合在 commit/recovery 位置增加 `interrupt_take`。

### 14.5 MRET

执行 `MRET`：

```text
PC = mepc
MIE = MPIE
MPIE = 1
按设计更新 MPP
清空年轻流水状态
```

需要 directed test 检查 MRET 后的第一条指令没有丢失或重复。

### 14.6 访存和 MMIO

CPU 核负责：

- 生成正确地址、数据和 `wstrb`；
- 普通 DRAM 可缓存；
- MMIO uncached；
- load 等待 `resp_valid`；
- MMIO 访问有序；
- 未对齐访问按设计 Trap 或正确拆分；
- 中断发生时不破坏未完成访问。

RT-Thread 按 ABI 产生的栈和普通对象大多自然对齐，但 packed 数据和 byte/halfword 访问仍要验证。

### 14.7 可选功能

```text
WFI
软件中断
PMP
用户模式
完整 RV32A
FPU 上下文
```

均不是第一个 RT-Thread 版本的必需条件。

## 15. CPU 核与 SoC 各自负责什么

| 功能 | CPU 核 | SoC |
|---|---|---|
| 执行 RV32 指令 | 负责 | 不负责 |
| 通用寄存器、流水线 | 负责 | 不负责 |
| CSR | 负责 | 提供 IRQ 输入 |
| 精确异常/中断 | 负责 | 产生外部事件 |
| `mret` | 负责 | 不负责 |
| IROM 地址和取指 | 发请求并接收指令 | 实现存储器和地址映射 |
| load/store | 发出数据请求 | DRAM/MMIO 译码和响应 |
| DCache | CPU 内部 | 不缓存 MMIO |
| UART 串行协议 | 不负责 | 负责 |
| Timer 计数和比较 | 不负责 | 负责 |
| 中断 pending 来源 | 在 `mip` 表示 | 外设产生 |
| LED/SEG | 发 MMIO store | 寄存并驱动输出 |
| PLL/复位同步 | 不负责 | 板级顶层负责 |
| COE 初始化 | 不负责 | FPGA IP/构建流程负责 |

“CPU 处理中断”与“Timer 产生中断”是两个不同职责。

## 16. `SocMemBridge` 推荐接口

可扩展为：

```systemverilog
module SocMemBridge #(
    parameter logic [31:0] P_DRAM_ADDR_START = 32'h8010_0000,
    parameter logic [31:0] P_DRAM_ADDR_END   = 32'h8014_0000
) (
    input  logic        clk,
    input  logic        rst,

    input  logic        req_valid,
    output logic        req_ready,
    input  logic        req_write,
    input  logic [31:0] req_addr,
    input  logic [31:0] req_wdata,
    input  logic [3:0]  req_wstrb,
    input  logic        req_uncached,
    output logic        resp_valid,
    output logic [31:0] resp_rdata,

    input  logic        uart_rx,
    output logic        uart_tx,
    output logic        irq_timer,
    output logic        irq_external,

    input  logic [63:0] virtual_sw_input,
    input  logic [7:0]  virtual_key_input,
    output logic [39:0] virtual_seg_output,
    output logic [31:0] virtual_led_output
);
```

如果 `display_seg` 继续用 50 MHz，额外保留 `cnt_clk/cnt_rst`，并只跨越显示数据，不跨越 CPU MMIO 总线。

## 17. SoC 顶层连接示意

```systemverilog
logic irq_software;
logic irq_timer;
logic irq_external;

assign irq_software = 1'b0;

myCPU Core_cpu (
    .cpu_rst          (cpu_rst_sync),
    .cpu_clk          (w_cpu_clk),

    .irom_addr        (irom_addr),
    .irom_data        (instruction),
    .irom_ena         (irom_ena),

    .dmem_req_valid   (dmem_req_valid),
    .dmem_req_ready   (dmem_req_ready),
    .dmem_req_write   (dmem_req_write),
    .dmem_req_addr    (dmem_req_addr),
    .dmem_req_wdata   (dmem_req_wdata),
    .dmem_req_wstrb   (dmem_req_wstrb),
    .dmem_req_uncached(dmem_req_uncached),
    .dmem_resp_valid  (dmem_resp_valid),
    .dmem_resp_rdata  (dmem_resp_rdata),

    .irq_software     (irq_software),
    .irq_timer        (irq_timer),
    .irq_external     (irq_external)
);

IROM_0 Mem_IROM (
    .addra(irom_word_addr),
    .clka (w_cpu_clk),
    .ena  (irom_ena),
    .douta(instruction)
);

SocMemBridge mem_bridge (
    .clk         (w_cpu_clk),
    .rst         (cpu_rst_sync),

    .req_valid   (dmem_req_valid),
    .req_ready   (dmem_req_ready),
    .req_write   (dmem_req_write),
    .req_addr    (dmem_req_addr),
    .req_wdata   (dmem_req_wdata),
    .req_wstrb   (dmem_req_wstrb),
    .req_uncached(dmem_req_uncached),
    .resp_valid  (dmem_resp_valid),
    .resp_rdata  (dmem_resp_rdata),

    .uart_rx     (uart_rx),
    .uart_tx     (uart_tx),
    .irq_timer   (irq_timer),
    .irq_external(irq_external)
);
```

这是结构示意，完整版本还要连接 GPIO、显示和 debug。

## 18. 时钟和复位

### 18.1 时钟域

当前：

```text
CPU/IROM/DRAM/core：cpu_clk
digital-twin/counter/display：50 MHz
```

新增 RT-Thread 外设建议：

```text
MMIO UART：cpu_clk
machine timer：cpu_clk
irq_ctrl：cpu_clk
```

这样中断和 MMIO 都不需 CDC。

### 18.2 复位

异步拉起、同步释放：

```systemverilog
always_ff @(posedge cpu_clk or negedge pll_locked) begin
    if (!pll_locked) begin
        rst_meta <= 1'b1;
        rst_sync <= 1'b1;
    end else begin
        rst_meta <= 1'b0;
        rst_sync <= rst_meta;
    end
end
```

每个时钟域使用自己的同步释放链。

### 18.3 CDC

跨域信号分类处理：

| 信号 | 方法 |
|---|---|
| 单 bit level | 两级同步 |
| pulse | toggle/握手 |
| 多 bit 配置、更新不频繁 | 发送端保持 + 同步 strobe |
| 高速数据流 | asynchronous FIFO |

不能给 32 位总线每一位各加两级同步器，就认为得到了同时一致的数据。

## 19. FPGA IP 与行为模型的分工

```text
Verilator
├─ rtl/ip/IROM_0.sv
├─ rtl/ip/DRAM_0.sv
├─ rtl/ip/pll.sv
└─ 行为模型

Vivado FPGA
├─ blk_mem_gen IROM_0
├─ blk_mem_gen DRAM_0
├─ clk_wiz pll
└─ 真实 IP
```

FPGA filelist 不能同时加入行为模型和同名 Vivado IP，否则会重复定义。

两边必须保持：

- 端口名称；
- 数据宽度；
- 地址宽度；
- 读延迟；
- byte write；
- READ_FIRST 模式；
- 初始镜像含义。

## 20. Vivado 工程需要修改什么

### 20.1 IP

- IROM depth 从 4096 改为 65536；
- IROM COE 改为软件构建输出；
- DRAM 保持 65536 word；
- DRAM COE 改为软件构建输出；
- 保持读延迟与行为模型一致。

### 20.2 RTL 文件

加入：

```text
rtl/soc/uart_mmio.sv
rtl/soc/machine_timer.sv
rtl/soc/irq_ctrl.sv（第二阶段）
rtl/soc/soc_addr_pkg.sv
```

修改：

```text
rtl/core/core_top.sv
rtl/core/myCPU.sv
rtl/core/commit/csr_file.sv
rtl/core/control/recovery_ctrl.sv
rtl/soc/student_top.sv
rtl/soc/SocMemBridge.sv
rtl/soc/top.sv
scripts/filelists/core.f
scripts/filelists/soc.f
```

### 20.3 约束

- CPU 时钟约束；
- 50 MHz 时钟约束；
- 两域 CDC 约束；
- UART 引脚；
- LED/SEG 引脚；
- 若修改顶层端口，同步更新 XDC。

## 21. RTL 验证计划

### 21.1 模块级

#### UART

```text
TX 波特率和 stop bit
RX 采样
STATUS
RX valid clear
IRQ enable/pending
连续字符
overrun
```

#### Timer

```text
mtime 递增
mtimecmp 高低 32 位
compare 前后 IRQ
更新 compare 后 IRQ 清除
复位
```

#### IRQ controller

```text
pending
enable
priority
claim
complete
同时多个 source
```

#### CSR

```text
mie/mip 读写
mstatus.MIE/MPIE
mcause interrupt bit
mepc
mret
```

### 21.2 CPU directed tests

```text
interrupt masked
interrupt disabled
timer interrupt entry
external interrupt entry
interrupt after CSR write
interrupt during load
interrupt around store
MRET
multiple pending
```

### 21.3 SoC 集成

```text
裸机 hello
.data/.bss/.rodata
UART loopback
Timer 1 ms
两个线程切换
FinSH
CoreMark
```

### 21.4 FPGA

同一份 ELF 生成的镜像在 Verilator 和 FPGA 上运行。比较：

- 第一条 PC；
- 启动日志；
- Tick 频率；
- UART 波特率；
- CoreMark CRC；
- 关键性能计数。

## 22. SoC 层需要实现什么

必须实现：

- 稳定板级顶层和 SoC 顶层；
- IROM 256 KiB 或明确的足够容量；
- DRAM 256 KiB；
- 双镜像初始化；
- MMIO 地址译码；
- CPU 可见 UART；
- machine timer；
- Timer IRQ；
- External IRQ；
- CPU 的 `mie/mip/mcause` 和精确中断；
- MRET；
- Verilator/Vivado 一致的存储器时序；
- filelist、Tcl 和 XDC 更新。

第二阶段：

- UART RX FIFO；
- 简化 IRQ controller/PLIC；
- `WFI`；
- `updatemem`；
- 运行时 loader。

不属于 SoC RTL：

- `link.lds`；
- `start.S`；
- RT-Thread 调度器；
- CoreMark 算法；
- `main.c`。

## 23. SoC 完成标准

- 复位入口固定且不会回绕；
- IROM/DRAM 容量与软件脚本一致；
- FPGA IP 和行为模型延迟一致；
- `.text` 与数据 section 从正确存储器读取；
- UART console 可双向通信；
- Timer IRQ 周期稳定；
- External IRQ 可屏蔽、可响应；
- `mepc/mcause/mstatus/mie/mip` 通过 directed tests；
- 未完成访存与中断边界行为明确；
- MMIO 不经过 Cache，不被重排或合并；
- Verilator 启动 RT-Thread；
- FPGA 出现 `msh >`；
- CoreMark CRC 通过；
- 修改接口后，RTL 回归、软件构建和 FPGA 工程均重新验证。

## 24. 第一轮实施顺序

```text
1. 冻结地址映射和端口
2. IROM 扩到 256 KiB
3. 验证 IMEM/DMEM 双镜像
4. 增加 UART MMIO，先做 polling TX
5. 增加 machine timer
6. CPU 增加 irq_timer/irq_external
7. CSR 增加 mie/mip 和 32 位 mcause
8. 实现精确中断和 MRET 回归
9. 裸机 Timer IRQ
10. RT-Thread 首线程与 Tick
11. UART RX/FinSH
12. CoreMark
13. 第二阶段增加 IRQ controller 和更新 bitstream 工具
```

这条顺序尽量避免在操作系统已经接入后再次大改底层接口。
