# RTL 内部层次与接口规划：从 CPU Core 到 SoC Top

> 本文只讨论 `rtl/` 内部怎样划分，以及各层之间用什么接口连接。  
> 重点回答：CPU 核放在 SoC 中时究竟有哪些输入输出，Cache 放在哪里，HXI 怎样连接 CPU、BRAM 和外设，Timer/UART/IRQ 又如何接入。  
> 本文是对 13 号文档的进一步聚焦：13 号偏仓库与工具组织，本文偏 RTL 模块架构和端口契约。

---

## 0. 先给出推荐的 RTL 总体结构

推荐的核心连接是：

```text
soc_top
│
├─ cpu_subsystem
│  ├─ cpu_core
│  │  ├─ frontend
│  │  ├─ decode / issue / execute / commit
│  │  ├─ lsu
│  │  └─ csr_trap
│  ├─ icache / fetch_buffer
│  ├─ dcache
│  ├─ store_buffer
│  ├─ HXI-I Master Port
│  └─ HXI-D Master Port
│
├─ hxi_crossbar
│  ├─ M0：CPU I-side
│  ├─ M1：CPU D-side
│  ├─ S0：Code BRAM
│  ├─ S1：Data BRAM
│  ├─ S2：Machine Timer
│  ├─ S3：Interrupt Controller
│  ├─ S4：HXI-to-APB Bridge
│  └─ S5：Default Error Slave
│
├─ memory_subsystem
│  ├─ hxi_code_mem_slave
│  ├─ code_bram_wrapper
│  ├─ hxi_data_mem_slave
│  └─ data_bram_wrapper
│
├─ machine_timer
│  └─ irq_timer → CPU
│
├─ interrupt_controller
│  ├─ UART/GPIO 等 IRQ 输入
│  └─ irq_external → CPU
│
└─ apb_subsystem
   ├─ hxi_to_apb
   ├─ apb_uart
   ├─ apb_gpio
   └─ 其他低速寄存器外设
```

CPU 面向 SoC 的接口不应再是：

```text
“IROM 的某几个特殊端口 + DRAM 的某几个特殊端口”
```

最终推荐统一成：

```text
时钟和复位
两组 HXI Master
三路机器模式中断
可选 Debug/Commit Trace
```

也就是：

```text
cpu_subsystem
├─ clk/rst
├─ I-HXI Master
├─ D-HXI Master
├─ irq_software
├─ irq_timer
├─ irq_external
└─ debug trace
```

---

## 1. 推荐的 `rtl/` 内部文件结构

```text
rtl/
├─ pkg/
│  ├─ soc_config_pkg.sv
│  ├─ memory_map_pkg.sv
│  ├─ hxi_pkg.sv
│  ├─ apb_pkg.sv
│  ├─ cpu_config_pkg.sv
│  └─ cpu_types_pkg.sv
│
├─ common/
│  ├─ reset_sync.sv
│  ├─ bit_sync.sv
│  ├─ pulse_sync.sv
│  ├─ async_fifo.sv
│  ├─ skid_buffer.sv
│  ├─ round_robin_arbiter.sv
│  └─ fifo_sync.sv
│
├─ cpu/
│  ├─ core/
│  │  ├─ frontend/
│  │  │  ├─ pc_control.sv
│  │  │  ├─ fetch_queue.sv
│  │  │  └─ branch_predictor.sv
│  │  ├─ decode/
│  │  ├─ issue/
│  │  ├─ execute/
│  │  ├─ lsu/
│  │  ├─ commit/
│  │  ├─ csr_trap/
│  │  │  ├─ csr_file.sv
│  │  │  ├─ irq_pending_ctrl.sv
│  │  │  ├─ trap_priority.sv
│  │  │  └─ trap_redirect.sv
│  │  ├─ control/
│  │  ├─ perf/
│  │  └─ cpu_core.sv
│  │
│  ├─ cache/
│  │  ├─ icache/
│  │  │  ├─ icache.sv
│  │  │  ├─ icache_tag_array.sv
│  │  │  └─ icache_data_array.sv
│  │  ├─ dcache/
│  │  │  ├─ dcache.sv
│  │  │  ├─ dcache_tag_array.sv
│  │  │  └─ dcache_data_array.sv
│  │  └─ cache_mem_arbiter.sv
│  │
│  ├─ memory_order/
│  │  ├─ store_buffer.sv
│  │  ├─ load_queue.sv
│  │  └─ mmio_order_ctrl.sv
│  │
│  ├─ bus_adapter/
│  │  ├─ ifu_to_hxi.sv
│  │  └─ lsu_to_hxi.sv
│  │
│  └─ cpu_subsystem.sv
│
├─ bus/
│  ├─ hxi_decoder.sv
│  ├─ hxi_arbiter.sv
│  ├─ hxi_req_router.sv
│  ├─ hxi_rsp_router.sv
│  ├─ hxi_owner_tracker.sv
│  ├─ hxi_default_slave.sv
│  └─ hxi_crossbar.sv
│
├─ memory/
│  ├─ hxi_code_mem_slave.sv
│  ├─ hxi_data_mem_slave.sv
│  ├─ code_bram_wrapper.sv
│  ├─ data_bram_wrapper.sv
│  └─ memory_subsystem.sv
│
├─ peripheral/
│  ├─ timer/
│  │  └─ machine_timer.sv
│  ├─ irq/
│  │  └─ interrupt_controller.sv
│  ├─ uart/
│  │  ├─ uart_tx.sv
│  │  ├─ uart_rx.sv
│  │  ├─ uart_fifo.sv
│  │  └─ apb_uart.sv
│  ├─ gpio/
│  │  └─ apb_gpio.sv
│  └─ legacy/
│     └─ stopwatch_counter.sv
│
├─ bridge/
│  ├─ hxi_to_apb.sv
│  └─ apb_interconnect.sv
│
└─ soc/
   ├─ irq_router.sv
   ├─ apb_subsystem.sv
   ├─ memory_subsystem_wrapper.sv
   └─ soc_top.sv
```

FPGA 板级顶层建议仍放在：

```text
fpga/.../fpga_top.sv
```

因为它包含差分时钟、PLL、Xilinx IP 和具体管脚，不属于通用 SoC RTL 内部结构。

---

## 2. 每一层只能依赖谁

推荐依赖方向：

```text
pkg
 ↑
common
 ↑
cpu / bus / memory / peripheral / bridge
 ↑
soc_top
 ↑
fpga_top 或 soc_sim_top
```

更具体地说：

```text
cpu_core
    可以依赖 cpu_types_pkg
    不允许依赖 UART、Timer、BRAM IP、Crossbar

cpu_subsystem
    可以依赖 cpu_core、Cache、StoreBuffer、HXI package
    不允许依赖具体 UART 地址判断

hxi_crossbar
    可以依赖 hxi_pkg、arbiter、decoder
    不允许依赖 CPU 内部 uop/ROB 类型

memory_subsystem
    可以依赖 HXI 和 BRAM wrapper
    不允许依赖 CPU Pipeline

peripheral
    可以依赖 APB/HXI package
    不允许读取 CPU 内部状态

soc_top
    可以实例化以上全部模块
    但自身不实现复杂功能状态机
```

这个依赖方向能保证：

- CPU 可以脱离 SoC 单独仿真；
- Crossbar 可以使用 Test Master 独立测试；
- Timer/UART 可以在没有 CPU 的情况下测试；
- 更换 CPU 核时，外设不需要重写；
- 更换 UART 时，CPU 不需要修改。

---

## 3. `cpu_core` 与 `cpu_subsystem` 的区别

### 3.1 `cpu_core`

`cpu_core` 负责 CPU 架构行为：

- 执行 RISC-V 指令；
- 维护通用寄存器；
- 产生取指请求；
- 产生 Load/Store 请求；
- 处理数据相关；
- 处理分支；
- CSR；
- 同步异常；
- 异步中断；
- 精确 Trap；
- `mret`；
- 指令提交。

它看到的是：

```text
“我要读取 PC 对应的指令”
“我要读取/写入某个物理地址”
“Timer IRQ 已经 pending”
```

它不应知道：

- 这个地址最终落在哪个 BRAM；
- UART 基地址是多少；
- HXI Crossbar 有几个 Slave；
- FPGA 使用什么 Block Memory Generator；
- UART TX 是哪一个 FPGA 引脚。

### 3.2 `cpu_subsystem`

`cpu_subsystem` 在 `cpu_core` 外面再包一层：

```text
cpu_subsystem
├─ cpu_core
├─ I-cache 或 Fetch Buffer
├─ D-cache
├─ Store Buffer
├─ MMIO 顺序控制
├─ IFU-to-HXI Adapter
└─ LSU-to-HXI Adapter
```

它负责把 CPU 内部请求变成 SoC 统一 HXI 请求。

### 3.3 为什么 Cache 放在 `cpu_subsystem`

Cache 的位置是：

```text
CPU Core
    ↓
Cache
    ↓
HXI Master
    ↓
SoC Crossbar
```

从处理器实现角度，Cache 属于处理器子系统；从 SoC 互联角度，它是 HXI Master 的上游。

因此：

- Cache 不与 UART/GPIO 放在 `peripheral/`；
- Cache 不放在 BRAM Slave 内；
- Crossbar 不需要知道一次访问是否 Cache Hit；
- MMIO 在 CPU 子系统内识别并绕过 D-cache。

---

## 4. CPU 核内部建议使用什么接口

`cpu_core` 和 Cache/Bus Adapter 之间可以使用更接近流水线语义的内部接口，不必直接使用完整 HXI。

### 4.1 IFU 内部接口

```text
Core → I-cache/Fetch：
    if_req_valid
    if_req_addr[31:0]
    if_req_epoch/tag

I-cache/Fetch → Core：
    if_req_ready
    if_rsp_valid
    if_rsp_data[31:0]
    if_rsp_fault
    if_rsp_epoch/tag
```

`epoch/tag` 用于区分分支 Redirect 以后返回的旧取指响应。

### 4.2 LSU 内部接口

```text
Core → LSU/Cache：
    mem_req_valid
    mem_req_write
    mem_req_addr[31:0]
    mem_req_wdata[31:0]
    mem_req_wstrb[3:0]
    mem_req_size[1:0]
    mem_req_sign
    mem_req_id

LSU/Cache → Core：
    mem_req_ready
    mem_rsp_valid
    mem_rsp_rdata[31:0]
    mem_rsp_fault
    mem_rsp_id
```

这些字段属于 CPU 内部实现。SoC 不需要知道：

- Load 是否有符号扩展；
- 该 Load 属于哪个 Scoreboard ID；
- 分支恢复 Epoch；
- ROB/提交编号。

CPU Bus Adapter 必须在边界把 CPU 内部字段转换为 HXI 字段。

---

## 5. `cpu_subsystem` 面向 SoC 的完整接口

这是本文最重要的接口。

推荐的职责级端口如下：

```systemverilog
module cpu_subsystem (
    input  logic        clk_i,
    input  logic        rst_i,

    // Instruction HXI master
    output logic        i_hxi_req_valid_o,
    input  logic        i_hxi_req_ready_i,
    output logic [31:0] i_hxi_req_addr_o,
    output logic        i_hxi_req_write_o,
    output logic [31:0] i_hxi_req_wdata_o,
    output logic [3:0]  i_hxi_req_wstrb_o,
    output logic [1:0]  i_hxi_req_size_o,
    output logic        i_hxi_req_instr_o,

    input  logic        i_hxi_rsp_valid_i,
    output logic        i_hxi_rsp_ready_o,
    input  logic [31:0] i_hxi_rsp_rdata_i,
    input  logic        i_hxi_rsp_err_i,

    // Data HXI master
    output logic        d_hxi_req_valid_o,
    input  logic        d_hxi_req_ready_i,
    output logic [31:0] d_hxi_req_addr_o,
    output logic        d_hxi_req_write_o,
    output logic [31:0] d_hxi_req_wdata_o,
    output logic [3:0]  d_hxi_req_wstrb_o,
    output logic [1:0]  d_hxi_req_size_o,
    output logic        d_hxi_req_instr_o,

    input  logic        d_hxi_rsp_valid_i,
    output logic        d_hxi_rsp_ready_o,
    input  logic [31:0] d_hxi_rsp_rdata_i,
    input  logic        d_hxi_rsp_err_i,

    // RISC-V machine-mode interrupts
    input  logic        irq_software_i,
    input  logic        irq_timer_i,
    input  logic        irq_external_i,

    // Stable commit/debug interface
    output logic        commit_valid_o,
    output logic [31:0] commit_pc_o,
    output logic [31:0] commit_inst_o,
    output logic        commit_wen_o,
    output logic [4:0]  commit_rd_o,
    output logic [31:0] commit_wdata_o,
    output logic        commit_trap_o,
    output logic [31:0] commit_cause_o
);
```

这只是接口骨架，不包含实现。

### 5.1 为什么 I-side 也保留完整 HXI 字段

I-side 实际永远：

```text
req_write = 0
req_wdata = 0
req_wstrb = 0
req_instr = 1
```

保留完整字段的好处是：

- Crossbar 所有 Master 使用同一种请求类型；
- 不需要专门设计一种 I-HXI；
- 连接数组更简单；
- Debug Loader 等新 Master 也能复用。

代价只是多几根固定信号，综合会优化掉常量逻辑。

如果你们更重视端口简洁，也可以在 `ifu_to_hxi.sv` 中补齐固定字段。两种方式都可以，但 Crossbar 内部必须统一。

### 5.2 `req_size`

建议：

```text
2'b00 = byte
2'b01 = halfword
2'b10 = word
2'b11 = reserved
```

I-side 固定为 word。

D-side 根据 LB/LH/LW/SB/SH/SW 产生。

### 5.3 `req_wstrb`

表示写入哪几个字节：

```text
0001  写 byte lane 0
0010  写 byte lane 1
0100  写 byte lane 2
1000  写 byte lane 3
0011  写低 halfword
1100  写高 halfword
1111  写整个 word
```

### 5.4 `rsp_err`

读写请求都必须有响应。

```text
读请求：
    rsp_rdata 有效

写请求：
    rsp_rdata 可忽略，但 rsp_valid 表示真正完成

rsp_err = 1：
    I-side → Instruction Access Fault
    D-side read → Load Access Fault
    D-side write → Store Access Fault
```

如果没有 `rsp_err`，非法地址只能：

- 永久等待；
- 或返回假数据；

两者都不适合完整 SoC。

### 5.5 三路 IRQ

```text
irq_software_i
    → mip.MSIP
    → mcause 0x8000_0003

irq_timer_i
    → mip.MTIP
    → mcause 0x8000_0007

irq_external_i
    → mip.MEIP
    → mcause 0x8000_000B
```

第一版没有 Software Interrupt 时：

```text
irq_software_i = 0
```

但接口可以预留。

### 5.6 Debug 接口为什么保持稳定

不要使用：

```systemverilog
`ifdef VERILATOR_TB
    才出现这些端口
`endif
```

建议端口始终存在：

- Verilator 连接；
- FPGA 不连接时由综合优化；
- 需要 ILA 时直接使用；
- Difftest 与 FPGA Trace 语义一致。

---

## 6. HXI 的握手契约

### 6.1 请求握手

只有在：

```text
req_valid && req_ready
```

同时为 1 的上升沿，请求才被接收。

如果：

```text
req_valid = 1
req_ready = 0
```

Master 必须保持：

```text
req_addr
req_write
req_wdata
req_wstrb
req_size
req_instr
```

不变。

### 6.2 响应握手

只有：

```text
rsp_valid && rsp_ready
```

时响应才被接收。

如果 CPU 暂时不能接收：

```text
rsp_valid = 1
rsp_ready = 0
```

Slave/Crossbar 必须保持：

```text
rsp_rdata
rsp_err
```

不变。

### 6.3 第一版 Outstanding

推荐：

```text
每个 Master 最多一笔请求已经发出但尚未收到响应
```

这样不需要：

- ID；
- 乱序响应；
- Response Reorder；
- 多事务 Scoreboard。

Cache Line refill 用四笔连续 word 请求完成，而不是第一版就做 burst。

### 6.4 不允许组合环

设计时要检查：

```text
Master valid 组合依赖 ready
Slave ready 又组合依赖 valid
```

是否形成环。

必要时在：

```text
CPU Master 边界
Crossbar Slave 边界
```

加入 Skid Buffer 或寄存切片。

---

## 7. Crossbar 的内部层次与接口

### 7.1 Crossbar 顶层

建议参数：

```text
NM = Master 数量，第一版为 2
NS = Slave 数量，第一版约 5～6
ADDR_WIDTH = 32
DATA_WIDTH = 32
```

接口可以用参数数组表达：

```text
Master request inputs  [NM]
Master request ready   [NM]
Master response outputs[NM]

Slave request outputs  [NS]
Slave request ready    [NS]
Slave response inputs  [NS]
```

### 7.2 `hxi_decoder`

对每个 Master 地址进行译码：

```text
M0 addr → onehot slave select
M1 addr → onehot slave select
```

未命中任何正常 Slave 时选择 Default Slave。

### 7.3 `hxi_arbiter`

每个 Slave 有自己的仲裁器。

例如：

```text
I-side ─┐
        ├→ Code BRAM Arbiter
D-side ─┘
```

不同 Master 访问不同 Slave 时可以并行。

### 7.4 `hxi_owner_tracker`

请求握手时保存：

```text
slave_owner[S] = 获胜 Master
slave_busy[S]  = 1
```

响应完成后：

```text
slave_busy[S] = 0
```

响应不能根据当前地址选择 Master，因为响应回来时地址早已可能变化。

### 7.5 `hxi_default_slave`

未映射访问返回：

```text
rsp_valid = 1
rsp_err   = 1
rsp_rdata = 固定调试值或 0
```

不能永远不响应。

---

## 8. 推荐的 Master 与 Slave 连接矩阵

第一版：

```text
Master
M0 = CPU I-side
M1 = CPU D-side

Slave
S0 = Code BRAM
S1 = Data BRAM
S2 = Machine Timer
S3 = Interrupt Controller
S4 = HXI-to-APB Bridge
S5 = Default Error Slave
```

连接矩阵：

| Master \ Slave | Code BRAM | Data BRAM | Timer | IRQ Ctrl | APB | Default |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| CPU I-side | 读 | 可选执行 | 禁止 | 禁止 | 禁止 | 是 |
| CPU D-side | 至少可读 | 读写 | 读写 | 读写 | 读写 | 是 |

### 8.1 为什么 D-side 要能读 Code BRAM

如果链接脚本将：

```text
.rodata
字符串常量
const 数组
```

放在 Code BRAM，它们由 Load 指令读取，走 D-side。

如果决定 `.rodata` 全部放 Data BRAM，则可以限制 D-side 不读 Code BRAM。但这必须与链接脚本一起冻结。

### 8.2 I-side 是否能从 Data BRAM 执行

第一版可以禁止，简化权限和仲裁。

如果以后支持：

- Bootloader 下载代码；
- RAM 中执行；
- 动态代码；

再允许 I-side 访问 Data BRAM，并处理 I-cache/FENCE.I。

---

## 9. Memory Subsystem 的 RTL 边界

### 9.1 `hxi_code_mem_slave`

负责：

- 接受 HXI 读请求；
- 检查只读权限；
- 地址减去 Code Base；
- 字节地址转 word address；
- 驱动 Code BRAM；
- 等待同步读延迟；
- 返回数据或 error。

它不负责：

- Cache；
- CPU 分支；
- ELF 加载；
- Vivado COE 生成。

### 9.2 `hxi_data_mem_slave`

负责：

- HXI 读写；
- `wstrb`；
- 地址转换；
- 对齐检查；
- BRAM 同步读延迟；
- 写完成响应；
- 越界 error。

### 9.3 BRAM Wrapper 接口

建议统一：

```text
输入：
    clk_i
    en_i
    addr_i
    we_i[3:0]
    wdata_i[31:0]

输出：
    rdata_o[31:0]
```

Code BRAM：

```text
we_i 固定 0
```

Data BRAM：

```text
支持 4 位 byte write enable
```

### 9.4 谁负责读延迟

BRAM Wrapper 明确声明：

```text
READ_LATENCY = 1 或 2
```

HXI Memory Slave 根据这个固定延迟产生响应。

不能：

```text
Verilator Model 是组合读
FPGA BRAM 是两拍读
但上层仍假设它们一样
```

Verilator Model 和 FPGA Wrapper 必须呈现相同可见延迟。

---

## 10. Machine Timer 的接口

Timer 推荐直接作为 HXI Slave。

### 10.1 总线侧

输入：

```text
hxi_req_valid_i
hxi_req_addr_i
hxi_req_write_i
hxi_req_wdata_i
hxi_req_wstrb_i
hxi_rsp_ready_i
```

输出：

```text
hxi_req_ready_o
hxi_rsp_valid_o
hxi_rsp_rdata_o
hxi_rsp_err_o
```

### 10.2 CPU 事件侧

输出：

```text
irq_timer_o
```

连接：

```text
machine_timer.irq_timer_o
→ cpu_subsystem.irq_timer_i
```

### 10.3 内部寄存器

```text
mtime[63:0]
mtimecmp[63:0]
```

比较：

```text
irq_timer_o = (mtime >= mtimecmp)
```

Timer 不直接访问 CPU CSR。它只输出 IRQ 电平；`mip.MTIP` 在 CPU 内部产生。

---

## 11. UART、GPIO 与 APB Subsystem

### 11.1 HXI-to-APB Bridge

上游是 HXI Slave：

```text
hxi_req_*
hxi_rsp_*
```

下游是 APB Master：

```text
paddr
psel
penable
pwrite
pwdata
pstrb
prdata
pready
pslverr
```

Bridge 内部 FSM：

```text
IDLE
→ APB_SETUP
→ APB_ACCESS
→ HXI_RESPONSE
```

### 11.2 `apb_interconnect`

输入一个 APB Master，输出多个 `PSEL`：

```text
PSEL_UART
PSEL_GPIO
PSEL_PERF
```

并将被选外设的：

```text
PRDATA
PREADY
PSLVERR
```

返回 Bridge。

### 11.3 UART 两类接口

APB 寄存器接口：

```text
CPU 配置/读写
```

物理串口接口：

```text
uart_rx_i
uart_tx_o
```

中断接口：

```text
uart_irq_o
```

连接：

```text
apb_uart.uart_irq_o
→ interrupt_controller.source_irq_i[UART_ID]
```

### 11.4 GPIO

总线侧：

```text
APB Slave
```

引脚侧：

```text
gpio_in_i
gpio_out_o
gpio_oe_o
```

中断侧：

```text
gpio_irq_o
```

---

## 12. Interrupt Controller 的接口

### 12.1 外设输入

```text
source_irq_i[NIRQ-1:0]
```

例如：

```text
bit 0 = UART
bit 1 = GPIO
bit 2 = SPI
```

### 12.2 CPU 输出

```text
irq_external_o
```

连接：

```text
interrupt_controller.irq_external_o
→ cpu_subsystem.irq_external_i
```

### 12.3 配置接口

可以：

- 直接作为 HXI Slave；
- 或放在 APB。

如果采用 PLIC 风格、寄存器窗口较大，独立 HXI Slave 较清晰。

第一版最简外部中断只有 UART 时，也可以暂时：

```text
uart_irq_o → irq_external_i
```

但端口和目录仍预留 `interrupt_controller`。

### 12.4 Timer 不接进 External IRQ Controller

```text
Timer → irq_timer_i → MTIP/cause 7

UART/GPIO → IRQ Controller
          → irq_external_i → MEIP/cause 11
```

两条路径不能混淆。

---

## 13. `soc_top` 应有哪些输入输出

`soc_top` 是厂商无关的 SoC 集成层。

推荐：

```systemverilog
module soc_top #(
    parameter int unsigned GPIO_IN_WIDTH  = 32,
    parameter int unsigned GPIO_OUT_WIDTH = 32
) (
    input  logic                       clk_i,
    input  logic                       rst_i,

    input  logic                       uart_rx_i,
    output logic                       uart_tx_o,

    input  logic [GPIO_IN_WIDTH-1:0]   gpio_in_i,
    output logic [GPIO_OUT_WIDTH-1:0]  gpio_out_o,
    output logic [GPIO_OUT_WIDTH-1:0]  gpio_oe_o,

    output logic                       debug_commit_valid_o,
    output logic [31:0]                debug_commit_pc_o,
    output logic [31:0]                debug_commit_inst_o,
    output logic                       debug_trap_o,
    output logic [31:0]                debug_cause_o
);
```

如果 Code/Data BRAM 在 `soc_top` 内部实例化，那么 SoC 顶层不需要暴露存储端口。

如果采用平台 Backend 注入，可以让 `soc_core` 暴露 Native Memory Port，再由：

```text
soc_top_generic
fpga_top
soc_sim_top
```

分别连接存储 Backend。

### 13.1 `soc_top` 内部只做连接

`soc_top` 可以：

- 实例化模块；
- 连接 HXI；
- 连接 IRQ；
- 传递参数；
- 连接 GPIO/UART。

不应直接写：

- Timer 计数；
- UART FSM；
- 地址译码大 case；
- Cache bypass；
- CPU Trap 优先级。

这些必须在各自 Owner 模块中。

---

## 14. `fpga_top` 面向开发板的接口

`fpga_top` 位于 SoC 外面。

典型端口：

```systemverilog
module fpga_top (
    input  logic        sys_clk_p_i,
    input  logic        sys_clk_n_i,
    input  logic        reset_button_i,
    input  logic        uart_rx_i,
    output logic        uart_tx_o,
    input  logic [N-1:0] switch_i,
    input  logic [M-1:0] key_i,
    output logic [K-1:0] led_o
);
```

内部：

```text
差分时钟 Buffer
→ PLL/MMCM
→ reset_sync
→ soc_top
```

如果保留 digital twin：

```text
digital_twin_wrapper
```

也应放在 FPGA/板级层，而不是 CPU 核或 HXI 内部。

---

## 15. SoC 内部的完整连线

```text
                       ┌───────────────────────┐
                       │     cpu_subsystem     │
                       │                       │
       Timer IRQ ─────→│ irq_timer            │
    External IRQ ─────→│ irq_external         │
                       │                       │
                       │ I-HXI        D-HXI    │
                       └───┬────────────┬──────┘
                           │ M0         │ M1
                           ▼            ▼
                    ┌──────────────────────────┐
                    │       hxi_crossbar       │
                    └─┬─────┬─────┬─────┬─────┘
                      │     │     │     │
             S0       │     │     │     │ S4
       ┌──────────────▼┐    │     │  ┌──▼──────────┐
       │ Code BRAM     │    │     │  │ HXI-to-APB │
       │ HXI Slave     │    │     │  └──┬──────────┘
       └───────────────┘    │     │     │
             S1             │     │     ▼
       ┌────────────────────▼┐    │  ┌──────────────┐
       │ Data BRAM HXI Slave │    │  │ APB UART    │──→ uart_tx
       └─────────────────────┘    │  │ APB GPIO    │──→ gpio
             S2                   │  └──────┬───────┘
       ┌──────────────────────────▼┐        │ IRQ
       │ Machine Timer             │        ▼
       │ mtime / mtimecmp          │  ┌──────────────┐
       └──────────────┬────────────┘  │ IRQ Ctrl     │
                      │ irq_timer     └──────┬───────┘
                      └───────────────────────┼──→ irq_external
                                              │
                                              └→ CPU
```

注意：

- HXI 是 CPU 访问寄存器和存储器的数据路径；
- IRQ 是外设主动通知 CPU 的事件路径；
- 两者同时存在，但职责不同。

---

## 16. 地址和属性由谁负责

### 16.1 `memory_map_pkg.sv`

集中定义：

```text
CODE_BASE / CODE_MASK
DATA_BASE / DATA_MASK
TIMER_BASE / TIMER_MASK
IRQ_BASE / IRQ_MASK
APB_BASE / APB_MASK
UART_BASE
GPIO_BASE
```

### 16.2 Crossbar 负责目标选择

```text
addr → 哪个 Slave
```

### 16.3 CPU Subsystem 负责 Cache 属性

```text
Code/Data RAM → cacheable
Timer/APB/IRQ → non-cacheable
```

不能只相信软件给一个 `uncached` 位。

硬件至少应根据地址强制：

```text
MMIO bypass Cache
```

### 16.4 Slave 负责局部偏移

例如 UART：

```text
全局地址 0x....0010
→ APB 选择 UART
→ UART 内部 offset = 0x10
```

不要让 UART 自己重新判断整个 32 位系统地址。

---

## 17. 时钟和复位接口

第一版推荐所有核心 SoC RTL 使用同一个：

```text
clk_i
rst_i
```

包括：

- CPU；
- Crossbar；
- BRAM Adapter；
- Timer；
- APB；
- UART 数字逻辑。

UART 波特率使用 Clock Enable，不必生成新的 UART Clock。

这样可以大幅减少 CDC。

如果 digital twin 必须工作在独立 50 MHz 域：

```text
digital twin 域
↔ async_fifo/handshake
↔ SoC 域
```

### 17.1 复位策略

FPGA 层：

```text
外部复位/PLL unlocked
→ 异步拉起
→ 每个时钟域同步释放
```

RTL 功能模块：

```text
使用已经同步好的 rst_i
```

BRAM 数据数组不全量复位，只复位控制状态和 Valid 位。

---

## 18. Package 与接口类型

### 18.1 编译顺序

```text
soc_config_pkg
memory_map_pkg
hxi_pkg
apb_pkg
cpu_config_pkg
cpu_types_pkg
common
cpu
bus
memory
peripheral
bridge
soc_top
```

具体 package 若互相依赖，应调整成无环顺序。

### 18.2 HXI 类型

可以在 `hxi_pkg.sv` 中定义：

```systemverilog
typedef struct packed {
    logic [31:0] addr;
    logic        write;
    logic [31:0] wdata;
    logic [3:0]  wstrb;
    logic [1:0]  size;
    logic        instr;
} hxi_req_t;

typedef struct packed {
    logic [31:0] rdata;
    logic        err;
} hxi_rsp_t;
```

但公开模块端口是否直接使用 struct，要先同时验证当前 Verilator 和 Vivado。

最稳妥的第一版仍是扁平端口；package 用于常量和内部组合。

---

## 19. 当前工程接口怎样演进

当前 `myCPU.sv` 大致是：

```text
IROM：
    irom_addr
    irom_data
    irom_ena

DMEM：
    dmem_req_valid/ready
    dmem_req_write
    dmem_req_addr
    dmem_req_wdata
    dmem_req_wstrb
    dmem_req_uncached
    dmem_resp_valid
    dmem_resp_rdata

IRQ：
    当前没有
```

目标变化：

### I-side

```text
irom_addr/data/ena
→ I-HXI request/response
```

增加：

- ready；
- response valid；
- response error；
- Crossbar 访问。

### D-side

保留现有 request/response 思路，补齐：

```text
d_hxi_req_size
d_hxi_rsp_ready
d_hxi_rsp_err
```

`dmem_req_uncached` 可保留为内部 Debug/属性输出，但最终 MMIO 属性应由统一 Memory Map 强制。

### IRQ

新增：

```text
irq_software_i
irq_timer_i
irq_external_i
```

### Debug

移除改变端口列表的 `VERILATOR_TB`，改为稳定 Commit Trace。

---

## 20. 推荐的实现顺序

### 第一步：冻结接口，不移动全部文件

先写：

- `hxi_pkg.sv`；
- `memory_map_pkg.sv`；
- CPU Subsystem 端口；
- HXI 握手规范；
- IRQ 规范。

### 第二步：D-side HXI 正式化

在现有数据接口上增加：

- `rsp_ready`；
- `rsp_err`；
- `req_size`；
- Default Slave；
- 协议断言。

先保持 IROM 接口不变。

### 第三步：建立 HXI Crossbar 和 BRAM Slave

先接：

```text
M0 = D-side
S0 = Data BRAM
S1 = Default Slave
```

再扩展为两个 Master。

### 第四步：I-side 转 HXI

```text
IFU/I-cache → HXI-I → Code BRAM
```

### 第五步：Machine Timer 与 CPU IRQ

并行完成：

```text
队员 A：machine_timer
队长：CPU MTIP/MTIE/Trap
```

再做裸机 Timer IRQ 联调。

### 第六步：HXI-to-APB、UART、GPIO

完成 CPU 可访问的 Console UART。

### 第七步：Interrupt Controller

UART/GPIO 增多以后再加入。

### 第八步：清理旧 Bridge 和专用接口

新路径稳定后删除：

- 旧 `SocMemBridge`；
- 旧 `perip_bridge`；
- 重复地址译码；
- 只服务旧接口的 Wrapper。

---

## 21. 编码前必须确认的开放问题

以下问题还没有全部冻结：

1. 第一版 I-side 是否立即改 HXI，还是先保留专用 IROM；
2. 是否实现 I-cache；
3. `.rodata` 放 Code BRAM 还是 Data BRAM；
4. Code/Data BRAM 各多大；
5. BRAM 对上层呈现 1 拍还是 2 拍读取；
6. Timer 和 IRQ Controller 是否都独立 HXI Slave；
7. UART 与 digital twin 是否共用一个物理串口；
8. 第一版是否保留两个时钟域；
9. Commit Trace 是扁平端口还是 packed struct；
10. `soc_top` 是否内部实例化 BRAM Backend。

本文的推荐是：

```text
I-side：
    接口先定义 HXI，迁移可以晚于 D-side

I-cache：
    接口预留，是否实例化由性能测试决定

.rodata：
    D-side 至少能读取其所在区域

BRAM latency：
    以 FPGA 实际 IP 行为为准，Verilator 模型匹配

Timer：
    独立 HXI Slave

IRQ Controller：
    预留独立 Slave；第一版 UART 可先直连

Clock：
    SoC 逻辑尽量单域

Debug：
    稳定扁平端口
```

---

## 22. 最终验收标准

### CPU 边界

- [ ] CPU 只有 I-HXI、D-HXI、IRQ 和 Debug 等稳定 SoC 接口；
- [ ] CPU 不实例化 UART/Timer/Xilinx IP；
- [ ] I/D 两侧都能处理 error；
- [ ] MMIO 强制绕过 Cache；
- [ ] IRQ 能形成正确 `mcause/mepc/mstatus`；
- [ ] Debug 端口不随仿真宏改变。

### HXI

- [ ] 请求/响应握手明确；
- [ ] 背压时 payload 保持；
- [ ] 写请求也有完成响应；
- [ ] 一 Master 一 outstanding；
- [ ] 不同 Slave 可并行；
- [ ] 同一 Slave 正确仲裁；
- [ ] 响应按 Owner 返回；
- [ ] 未映射地址返回 error。

### Memory

- [ ] HXI Slave 与 BRAM Wrapper 分开；
- [ ] Verilator/FPGA 读取延迟一致；
- [ ] Data BRAM 支持 byte strobe；
- [ ] D-side 能访问 `.rodata`；
- [ ] RAM 数据数组不被全量复位。

### Peripheral

- [ ] Timer 同时有 HXI 寄存器接口和独立 IRQ；
- [ ] UART 同时有 APB、物理 RX/TX 和 IRQ；
- [ ] 外部 IRQ 与 Timer IRQ 分开；
- [ ] 外设不依赖 CPU 内部类型。

### SoC Top

- [ ] `soc_top` 只做实例化、连接和参数传递；
- [ ] 地址只在 `memory_map_pkg` 集中定义；
- [ ] FPGA PLL/管脚不进入通用 `soc_top`；
- [ ] 所有跨时钟域有明确 Owner；
- [ ] 每个模块可以在没有完整 SoC 时独立测试。

---

## 23. 一句话理解各层边界

```text
cpu_core：
    决定指令怎样执行

cpu_subsystem：
    把 CPU、Cache 和 HXI Master 组合成处理器

hxi_crossbar：
    决定一笔访问送给哪个设备、响应回给谁

memory_subsystem：
    把 HXI 访问变成 BRAM 读写

peripheral：
    提供 UART、Timer、GPIO 等具体功能

soc_top：
    把所有模块连成一台完整计算机

fpga_top：
    把这台计算机接到具体 FPGA 的时钟、IP 和管脚
```

对你当前项目，最关键的 CPU—SoC 边界就是：

```text
2 个 HXI Master
+ 3 路 IRQ
+ 1 组 Clock/Reset
+ 可选 Commit Trace
```

一旦这个边界冻结，CPU、SoC RTL 和 BSP 三部分才能真正并行推进。

