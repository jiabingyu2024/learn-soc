# RTL 项目结构规划：清晰、完整、Verilator 友好、FPGA 友好

> 适用项目：`superScalar` 自研 RV32 CPU、HXI SoC、FPGA BRAM、RT-Thread、FinSH 与 CoreMark。  
> 本文规划的是下一阶段推荐结构，不要求一次性移动全部现有文件。  
> 目标是让同一套可综合 RTL 同时服务 CPU 仿真、SoC 仿真和 FPGA 上板，并把仿真模型与 Xilinx IP 隔离在设计边界之外。

---

## 0. 最终建议

推荐把仓库分成七类内容：

```text
rtl/         厂商无关、可综合的设计 RTL
tb/          Testbench、Verilator C++ Harness、仿真模型
sim/         各仿真器配置、filelist、waiver 和运行入口
fpga/        板级顶层、XDC、Vivado Tcl 和 FPGA IP Wrapper
software/    裸机、BSP、RT-Thread、应用和链接脚本
scripts/     通用构建、镜像转换、结果分析工具
build/       全部生成物，不作为设计源
```

最重要的设计原则：

```text
rtl/ 中的 CPU、总线、Cache、Timer、UART 等功能逻辑
不能依赖 Verilator，也不能依赖 Xilinx IP 名称。

Verilator 模型只放 tb/models/。
Vivado IP 和板级逻辑只放 fpga/。
二者通过相同的 Wrapper/Backend 接口接入设计。
```

推荐保留三个顶层：

```text
cpu_sim_top       CPU ISA、异常、Cache 和性能测试
soc_sim_top       完整 SoC、UART、Timer、RT-Thread 仿真
fpga_top          PLL、引脚、Xilinx BRAM/IP 和真实上板
```

它们共享：

```text
同一个 cpu_subsystem
同一个 soc_core
同一个 HXI
同一个外设 RTL
```

区别只在测试环境和 FPGA 平台资源。

---

## 1. 为什么当前结构需要继续整理

当前工程已有：

```text
rtl/core/                  CPU 功能 RTL
rtl/soc/                   SoC、板级顶层和外设混放
rtl/ip/                    Verilator 使用的 FPGA IP 行为模型
scripts/filelists/         core/soc/Verilator filelist
fpga/create_vivado_project.tcl
tb/
build/
```

这些基础并不差：

- `core.f` 已按 package 优先排列；
- Verilator 有 CPU 和 `student_top` 两种构建；
- `ip_verilator.f` 已与 FPGA 真实 IP 区分；
- Vivado 工程可以由 Tcl 重建；
- `build/` 已用于生成物；
- Verilator 和 Vivado 共用主要 RTL。

但还有几个结构问题。

### 1.1 `rtl/soc/` 同时承担太多职责

当前其中同时有：

```text
板级 top
SoC integration
存储桥
BRAM adapter
UART
digital twin
Counter
数码管
```

这会导致：

- 不清楚哪些属于真正 SoC；
- 不清楚哪些只适用于当前比赛板；
- 仿真模型和可综合外设混淆；
- 后续 HXI/APB 重构时影响范围过大。

### 1.2 `rtl/ip/` 这个名字容易产生误解

里面当前主要是：

```text
IROM_0.sv
DRAM_0.sv
MUL_0.sv
DIV_0.sv
pll.sv
```

它们是 Verilator 行为模型，不是真正的 Vivado IP 输出。

因此更准确的位置是：

```text
tb/models/xilinx_compat/
```

真正的 FPGA IP 应由：

```text
fpga/ip/
fpga/tcl/
```

负责。

### 1.3 板级顶层与 SoC 顶层混在一起

当前：

```text
top.sv          差分时钟、PLL、物理 UART、digital twin
student_top.sv  CPU、IROM、数据桥
```

未来建议明确成：

```text
fpga_top.sv     只负责开发板和 FPGA IP
soc_core.sv     只负责芯片内部系统
soc_sim_top.sv  只负责完整 SoC 仿真环境
```

### 1.4 当前调试端口受 `VERILATOR_TB` 宏控制

条件编译改变模块端口列表会带来：

- Verilator 和 Vivado 看到不同的模块接口；
- 实例化处到处需要 `ifdef`；
- 文件顺序或宏遗漏时出现难懂错误；
- 以后增加另一个仿真器更麻烦。

更推荐：

1. 调试输出作为稳定端口存在，FPGA 未连接时由综合优化；
2. 或建立独立 `cpu_trace_wrapper`；
3. 或使用 `bind`/专用 Probe 模块；
4. 条件编译只控制 Probe 内部的仿真专用逻辑，不改变核心公开端口。

---

## 2. 推荐的完整仓库结构

```text
superScalar/
├─ rtl/
│  ├─ common/
│  │  ├─ pkg/
│  │  │  ├─ soc_config_pkg.sv
│  │  │  ├─ memory_map_pkg.sv
│  │  │  └─ common_types_pkg.sv
│  │  ├─ sync/
│  │  │  ├─ reset_sync.sv
│  │  │  ├─ bit_sync.sv
│  │  │  ├─ pulse_sync.sv
│  │  │  └─ async_fifo.sv
│  │  └─ util/
│  │     ├─ skid_buffer.sv
│  │     ├─ round_robin_arbiter.sv
│  │     └─ priority_encoder.sv
│  │
│  ├─ cpu/
│  │  ├─ pkg/
│  │  │  ├─ cpu_config_pkg.sv
│  │  │  └─ cpu_types_pkg.sv
│  │  ├─ frontend/
│  │  ├─ decode/
│  │  ├─ issue/
│  │  ├─ execute/
│  │  ├─ memory/
│  │  │  ├─ lsu/
│  │  │  ├─ store_buffer/
│  │  │  └─ load_queue/
│  │  ├─ cache/
│  │  │  ├─ icache/
│  │  │  └─ dcache/
│  │  ├─ commit/
│  │  ├─ csr_trap/
│  │  ├─ control/
│  │  ├─ perf/
│  │  ├─ debug/
│  │  ├─ cpu_core.sv
│  │  └─ cpu_subsystem.sv
│  │
│  ├─ bus/
│  │  ├─ hxi/
│  │  │  ├─ hxi_pkg.sv
│  │  │  ├─ hxi_skid_buffer.sv
│  │  │  ├─ hxi_decoder.sv
│  │  │  ├─ hxi_arbiter.sv
│  │  │  ├─ hxi_req_router.sv
│  │  │  ├─ hxi_rsp_router.sv
│  │  │  ├─ hxi_default_slave.sv
│  │  │  └─ hxi_crossbar.sv
│  │  └─ bridge/
│  │     ├─ hxi_to_apb.sv
│  │     └─ apb_interconnect.sv
│  │
│  ├─ memory/
│  │  ├─ interface/
│  │  │  └─ mem_native_pkg.sv
│  │  ├─ adapter/
│  │  │  ├─ hxi_code_mem_slave.sv
│  │  │  └─ hxi_data_mem_slave.sv
│  │  ├─ generic/
│  │  │  ├─ generic_rom.sv
│  │  │  ├─ generic_spram.sv
│  │  │  └─ generic_tdpram.sv
│  │  └─ memory_subsystem.sv
│  │
│  ├─ peripheral/
│  │  ├─ timer/
│  │  │  ├─ machine_timer.sv
│  │  │  └─ timer_reg_if.sv
│  │  ├─ irq/
│  │  │  └─ interrupt_controller.sv
│  │  ├─ uart/
│  │  │  ├─ uart_tx.sv
│  │  │  ├─ uart_rx.sv
│  │  │  ├─ uart_fifo.sv
│  │  │  └─ apb_uart.sv
│  │  ├─ gpio/
│  │  │  └─ apb_gpio.sv
│  │  └─ perf_ctrl/
│  │     └─ apb_perf_ctrl.sv
│  │
│  └─ soc/
│     ├─ cpu_soc_wrapper.sv
│     ├─ soc_core.sv
│     └─ soc_top_generic.sv
│
├─ tb/
│  ├─ common/
│  │  ├─ clock_reset_driver.sv
│  │  ├─ hxi_bfm.sv
│  │  ├─ apb_bfm.sv
│  │  └─ scoreboard/
│  ├─ unit/
│  │  ├─ cpu/
│  │  ├─ hxi/
│  │  ├─ cache/
│  │  ├─ memory/
│  │  ├─ timer/
│  │  ├─ uart/
│  │  └─ irq/
│  ├─ models/
│  │  ├─ generic/
│  │  │  ├─ rom_model.sv
│  │  │  ├─ ram_model.sv
│  │  │  └─ uart_host_model.sv
│  │  └─ xilinx_compat/
│  │     ├─ IROM_0.sv
│  │     ├─ DRAM_0.sv
│  │     ├─ MUL_0.sv
│  │     ├─ DIV_0.sv
│  │     └─ pll.sv
│  ├─ tops/
│  │  ├─ cpu_sim_top.sv
│  │  ├─ soc_sim_top.sv
│  │  └─ unit_sim_top.sv
│  ├─ cpp/
│  │  ├─ common/
│  │  ├─ cpu_main.cpp
│  │  └─ soc_main.cpp
│  └─ tests/
│     ├─ isa/
│     ├─ baremetal/
│     ├─ rtthread/
│     └─ coremark/
│
├─ sim/
│  ├─ filelists/
│  │  ├─ pkg.f
│  │  ├─ common_rtl.f
│  │  ├─ cpu_rtl.f
│  │  ├─ bus_rtl.f
│  │  ├─ memory_rtl.f
│  │  ├─ peripheral_rtl.f
│  │  ├─ soc_rtl.f
│  │  ├─ verilator_cpu.f
│  │  ├─ verilator_soc.f
│  │  └─ fpga_kintex7.f
│  ├─ verilator/
│  │  ├─ verilator_common.flags
│  │  ├─ verilator_lint.flags
│  │  ├─ verilator_waiver.vlt
│  │  └─ README.md
│  ├─ xsim/
│  └─ regression/
│     ├─ testlist.yaml
│     └─ expected_results.yaml
│
├─ fpga/
│  ├─ common/
│  │  ├─ rtl/
│  │  │  ├─ xilinx_code_mem_backend.sv
│  │  │  ├─ xilinx_data_mem_backend.sv
│  │  │  ├─ xilinx_mul_backend.sv
│  │  │  └─ xilinx_div_backend.sv
│  │  └─ tcl/
│  │     ├─ create_ip.tcl
│  │     ├─ read_rtl_filelist.tcl
│  │     └─ reports.tcl
│  ├─ boards/
│  │  └─ kintex7_competition/
│  │     ├─ rtl/
│  │     │  ├─ fpga_top.sv
│  │     │  └─ digital_twin_wrapper.sv
│  │     ├─ constraints/
│  │     │  ├─ pins.xdc
│  │     │  ├─ clocks.xdc
│  │     │  └─ cdc.xdc
│  │     ├─ tcl/
│  │     │  ├─ create_project.tcl
│  │     │  ├─ synth.tcl
│  │     │  └─ impl.tcl
│  │     └─ README.md
│  └─ profiles/
│     ├─ srcWithMext/
│     └─ rtthread/
│
├─ software/
│  ├─ startup/
│  ├─ bsp/
│  ├─ rt-thread/
│  ├─ applications/
│  ├─ coremark/
│  ├─ linker/
│  └─ build/
│
├─ scripts/
│  ├─ run_verilator.py
│  ├─ run_regression.py
│  ├─ elf2mem.py
│  ├─ check_filelists.py
│  ├─ check_memory_map.py
│  └─ collect_reports.py
│
├─ docs/
│  ├─ architecture/
│  ├─ interfaces/
│  ├─ memory_map/
│  ├─ verification/
│  └─ fpga/
│
├─ data/
│  ├─ images/
│  ├─ isa/
│  └─ profiles/
│
├─ build/
│  ├─ verilator/
│  ├─ xsim/
│  ├─ vivado/
│  ├─ software/
│  ├─ result/
│  └─ log/
│
├─ Makefile
├─ README.md
└─ .gitignore
```

这是目标结构。当前项目不需要立刻创建所有空目录；只有对应功能开始实现时才建立。

---

## 3. `rtl/` 的规则：只放设计本体

进入 `rtl/` 的文件应满足：

```text
可综合
不依赖 C++ Testbench
不依赖 Verilator 专有 API
不硬编码当前开发板管脚
不直接包含 XDC
不依赖某个 build/ 目录中的生成文件
不依赖绝对路径
```

### 3.1 允许出现什么

- SystemVerilog package；
- CPU；
- Cache；
- HXI/APB 逻辑；
- BRAM Adapter；
- 通用推断 RAM；
- UART/Timer/GPIO RTL；
- Reset/CDC 通用模块；
- 可综合断言或被工具忽略的仿真断言。

### 3.2 不允许出现什么

- C++ Harness；
- Verilator `main()`；
- Vivado `.xci/.dcp`；
- XDC；
- PLL 生成脚本；
- 纯 Testbench `initial` 激励；
- 只为了测试而存在的 UART Host；
- 仿真结果文件；
- 波形；
- ELF/COE 生成物。

### 3.3 不要在功能 RTL 中判断仿真器

应尽量避免：

```systemverilog
`ifdef VERILATOR
    一套功能行为
`else
    另一套功能行为
`endif
```

因为这样 Verilator 验证的可能不是 FPGA 真正综合的逻辑。

正确做法：

```text
功能 RTL 保持相同
目标相关差异放在 Backend/Wrapper
不同目标由 filelist 选择不同 Backend
```

---

## 4. CPU 目录怎样组织

### 4.1 `cpu_core.sv`

只负责处理器核心逻辑：

```text
Frontend
Decode
Issue
Execute
Commit
CSR/Trap
LSU
```

它不应实例化：

- UART；
- Timer；
- Xilinx BRAM；
- PLL；
- FPGA 引脚；
- digital twin。

### 4.2 `cpu_subsystem.sv`

推荐负责：

```text
cpu_core
I-cache/Fetch Queue
D-cache
Store Buffer
I-side HXI Adapter
D-side HXI Adapter
IRQ 输入整理
性能/Trace 边界
```

对 SoC 来说，`cpu_subsystem` 是一个有两个 Master 端口和若干 IRQ 输入的完整处理器子系统。

### 4.3 Cache 目录从 `memory/` 中分出

当前 `rtl/core/memory/` 同时包含：

- D-cache；
- LSU 辅助模块；
- Store Buffer；
- Load Queue；
- Memory Arbiter。

后续建议分成：

```text
cpu/memory/      指令语义和 Load/Store 控制
cpu/cache/       Cache tag/data/refill/write policy
```

原因：

- LSU 属于 CPU 执行语义；
- Cache 属于层次化存储系统；
- 两者验证重点不同；
- 后续添加 I-cache 时目录更清晰。

### 4.4 `csr_trap/` 独立

建议把：

```text
CSR File
IRQ Pending
Trap Priority
Trap Entry
MRET
```

集中到 `csr_trap/`，因为 RT-Thread 移植会频繁检查这一边界。

---

## 5. HXI 总线目录怎样组织

推荐拆成小而明确的模块：

```text
hxi_pkg.sv
    字段宽度、枚举、错误类型

hxi_decoder.sv
    地址 → Slave select

hxi_arbiter.sv
    多 Master 争用一个 Slave

hxi_req_router.sv
    请求送到获选 Slave

hxi_rsp_router.sv
    响应送回事务 Owner

hxi_default_slave.sv
    未映射地址返回 error

hxi_crossbar.sv
    实例化以上模块并维护 ownership
```

### 5.1 第一版接口不要过度复杂

建议：

```text
32 位地址
32 位数据
4 位 byte strobe
单拍请求
单拍响应
每 Master 最多一个 outstanding
响应按请求顺序
明确 error
Round-Robin 仲裁
```

暂不加入：

- burst；
- transaction ID；
- 乱序响应；
- 多 outstanding；
- Cache coherence。

### 5.2 端口表达方式

为了兼容不同版本的 Verilator 和 Vivado，公开集成边界优先使用明确的扁平端口：

```text
m0_req_valid
m0_req_ready
m0_req_addr
...
```

或使用参数化数组：

```text
req_valid_i[NM]
req_addr_i[NM][31:0]
```

SystemVerilog `interface/modport` 可以减少连线，但如果当前 Vivado/Verilator 版本组合没有充分验证，不要在重构第一阶段全面采用。

`typedef struct packed` 适合模块内部和 package，但要先做一次双工具编译测试。

---

## 6. 存储系统怎样同时适配 Verilator 和 FPGA

这是整个结构中最关键的边界。

### 6.1 分成三层

```text
HXI Memory Slave
    负责总线协议、地址检查和响应
           ↓
Native Memory Port
    统一的简单存储接口
           ↓
Memory Backend
    Verilator Model / Generic RTL / Xilinx IP Wrapper
```

### 6.2 Native Memory Port

建议统一为：

```text
clk
en
addr
we[3:0]
wdata[31:0]
rdata[31:0]
```

并冻结：

- 地址是字节地址还是 word 地址；
- 同步读取延迟为 1 拍还是 2 拍；
- Read-During-Write 行为；
- byte write；
- 输出是否额外寄存；
- reset 是否影响输出。

### 6.3 三种 Backend

#### Generic Synthesizable Backend

```text
rtl/memory/generic/
```

使用数组和 BRAM 推断风格，可同时用于：

- Verilator；
- Vivado 推断；
- 其他 FPGA。

优点是最接近“同一份 RTL”。

#### Verilator Fast Model

```text
tb/models/generic/
```

可提供：

- 快速镜像加载；
- 可配置延迟；
- 随机 Backpressure；
- 越界检查；
- 调试访问。

它必须遵守相同 Native Memory Port 行为。

#### Xilinx IP Backend

```text
fpga/common/rtl/
```

只负责把 Native Memory Port 映射到：

```text
IROM_0
DRAM_0
```

Xilinx 模块名不能出现在 CPU、HXI Crossbar 或 Cache 内。

### 6.4 不要复位 RAM 数据数组

如果对整个 RAM 数组逐项复位：

- Vivado 可能无法推断 BRAM；
- 生成大量触发器或复位逻辑；
- 时序和资源恶化。

正确方式：

```text
程序初值由 MEM/HEX/COE/$readmemh 提供
控制状态和 Valid 位复位
RAM 数据本体通常不复位
```

---

## 7. 外设目录怎样组织

每个外设目录至少包含：

```text
功能 datapath/FSM
总线寄存器接口
中断生成
必要 FIFO
对应单元测试
寄存器文档
```

### 7.1 UART

```text
uart_tx.sv       串行发送
uart_rx.sv       串行接收
uart_fifo.sv     字节缓冲
apb_uart.sv      APB 寄存器与 IRQ
```

不要把：

```text
物理 UART
digital-twin 协议
RT-Thread Console 寄存器
```

全部塞进一个 `uart.sv`。

### 7.2 Timer

```text
machine_timer.sv
    mtime、mtimecmp、IRQ

timer_reg_if.sv
    HXI/APB 寄存器访问
```

项目较小时可以合并成一个文件，但职责仍要在文档中区分。

### 7.3 Interrupt Controller

独立负责：

```text
外设 IRQ 输入
pending
enable
priority
claim/complete
CPU external IRQ
```

Timer IRQ 不应通过它。

---

## 8. 三个顶层分别负责什么

### 8.1 `cpu_sim_top.sv`

用途：

- RV32 ISA；
- CSR/Trap；
- Cache directed test；
- Difftest；
- IPC/性能分析。

结构：

```text
cpu_sim_top
├─ cpu_subsystem
├─ instruction_memory_model
├─ data_memory_model
└─ commit_trace/debug ports
```

它不实例化：

- APB UART；
- GPIO；
- 完整 Timer；
- FPGA PLL。

CPU 测试保持小、编译快。

### 8.2 `soc_sim_top.sv`

用途：

- 完整裸机；
- UART；
- Timer IRQ；
- RT-Thread；
- FinSH；
- CoreMark；
- SoC 回归。

结构：

```text
soc_sim_top
├─ soc_core
├─ simulation memory backend
├─ uart_host_model
├─ external pin model
└─ debug/termination monitor
```

### 8.3 `fpga_top.sv`

用途：

- 当前 Kintex-7 板；
- 差分时钟；
- PLL/MMCM；
- Reset Synchronizer；
- Xilinx BRAM/MUL/DIV IP；
- UART/GPIO 管脚；
- digital twin（如果保留）。

结构：

```text
fpga_top
├─ clock/reset platform logic
├─ soc_core
├─ xilinx memory backends
├─ xilinx arithmetic backends
└─ board I/O wrappers
```

### 8.4 为什么不直接用同一个 `top.sv`

因为三个目标的环境不同：

| 目标 | 时钟 | 存储 | 外部接口 | 关注点 |
| --- | --- | --- | --- | --- |
| CPU Sim | C++ 驱动 | 快速 Model | Commit Trace | CPU 正确性和性能 |
| SoC Sim | C++/SV 驱动 | SoC Model | UART Host | OS 和外设 |
| FPGA | PLL | Xilinx BRAM/IP | 真实 Pins | 综合、时序和上板 |

共享的是 `soc_core`，不是最外层环境。

---

## 9. Filelist 应怎样组织

### 9.1 Filelist 是唯一源码清单

Verilator、Vivado 和其他工具应尽量读取同一组层级化清单。

不要：

- 在 Vivado Tcl 中递归搜全部 `.sv`；
- 在 Verilator 脚本里维护另一套源码顺序；
- 同时依赖 glob 和 filelist；
- 把 `rtl/ip` 行为模型误加进 FPGA。

### 9.2 推荐层级

```text
pkg.f
    所有 package，按依赖顺序

common_rtl.f
    reset、sync、fifo、arbiter

cpu_rtl.f
    CPU，不重复包含 pkg.f

bus_rtl.f
memory_rtl.f
peripheral_rtl.f
soc_rtl.f
```

目标清单：

```text
verilator_cpu.f
├─ pkg.f
├─ common_rtl.f
├─ cpu_rtl.f
├─ CPU 仿真模型
└─ cpu_sim_top

verilator_soc.f
├─ pkg.f
├─ common_rtl.f
├─ cpu_rtl.f
├─ bus_rtl.f
├─ memory_rtl.f
├─ peripheral_rtl.f
├─ soc_rtl.f
├─ SoC 仿真模型
└─ soc_sim_top

fpga_kintex7.f
├─ pkg.f
├─ common_rtl.f
├─ cpu_rtl.f
├─ bus_rtl.f
├─ memory_rtl.f
├─ peripheral_rtl.f
├─ soc_rtl.f
├─ Xilinx Backend Wrapper
└─ fpga_top
```

### 9.3 明确 `--top-module`

Verilator 构建时必须显式指定：

```text
--top-module cpu_sim_top
```

或：

```text
--top-module soc_sim_top
```

不要依赖工具猜测 Top，否则文件中存在多个未实例化模块时容易出现 `MULTITOP`。

### 9.4 路径全部相对仓库根目录

Filelist 中使用：

```text
rtl/cpu/cpu_core.sv
```

不要写个人电脑绝对路径。

运行脚本先定位 repo root，再把 filelist 交给工具。

### 9.5 自动检查

建议 `scripts/check_filelists.py` 检查：

- 文件是否存在；
- 是否重复；
- package 是否在使用者之前；
- FPGA 清单是否包含 `tb/models`；
- Verilator 清单是否误包含 `.xci/.dcp`；
- 同一模块是否有多个定义；
- 顶层是否存在。

---

## 10. Verilator 友好的具体规则

### 10.1 设计 RTL 不使用延时

不要在 `rtl/` 中写：

```systemverilog
#5
wait (...)
fork/join
```

时钟、复位和激励由 Testbench/C++ Harness 负责。

### 10.2 避免仿真依赖深层层次路径

不推荐 C++ 直接长期依赖：

```text
TOP.dut.u_core.u_cache.data_array[...]
```

目录或模块一重构，Harness 就全部失效。

推荐通过稳定接口：

- Commit Trace Port；
- Debug Memory Loader；
- UART Host Model；
- DPI debug API；
- 顶层显式调试端口。

### 10.3 模型可配置延迟和 Backpressure

SoC Model 不应永远零等待。

建议参数：

```text
READ_LATENCY
WRITE_LATENCY
READY_STALL_PROBABILITY
ERROR_ADDRESS_RANGE
```

默认使用确定值，随机压力测试使用固定 seed，保证问题可重现。

### 10.4 波形默认关闭

普通回归：

```text
不生成波形
```

失败复现：

```text
TRACE=1
优先 FST
限制 trace depth
```

否则完整 RT-Thread/CoreMark 会产生巨大波形。

### 10.5 Warning 策略

推荐：

```text
新 Warning 默认视为错误
已确认的工具限制写入 waiver.vlt
waiver 必须注明原因和文件范围
禁止全局关闭 WIDTH/LATCH/MULTIDRIVEN
```

### 10.6 Build 目录按目标隔离

```text
build/verilator/cpu/
build/verilator/soc/
build/verilator/unit/timer/
```

不能让两个 Top 共用同一个 `obj_dir`。

Verilator 官方支持用 `--Mdir` 指定输出目录，并用 `--top-module` 明确顶层，适合这种多目标结构。

---

## 11. FPGA 友好的具体规则

### 11.1 Vivado 工程必须可由 Tcl 重建

继续保留当前方向：

```text
源码 + filelist + XDC + Tcl + IP 参数
→ create_project.tcl
→ 可重建 .xpr
```

不要把生成的整个 `.xpr/.runs/.cache` 当成唯一工程源。

### 11.2 FPGA IP 只在平台目录出现

例如：

```text
fpga/common/rtl/xilinx_data_mem_backend.sv
```

它可以实例化：

```text
DRAM_0
```

但：

```text
dcache.sv
hxi_crossbar.sv
machine_timer.sv
```

不应知道 `DRAM_0` 的名字。

### 11.3 XDC 按职责拆分

```text
pins.xdc
    引脚和 I/O standard

clocks.xdc
    输入时钟和 generated clock

cdc.xdc
    合理的异步时钟组和同步器约束

debug.xdc
    可选 ILA/debug 约束
```

不要把所有约束放在一个难以维护的大文件中。

### 11.4 FPGA 顶层只做平台集成

`fpga_top.sv` 可以有：

- PLL；
- 复位同步；
- Xilinx Backend；
- 管脚连接；
- ILA；

但不应在顶层临时加入：

- HXI 地址译码；
- UART 寄存器；
- Timer 比较逻辑；
- Cacheable 判断。

这些行为属于 `rtl/` 中的 Owner 模块。

### 11.5 保持推断友好

- 时序逻辑使用 `always_ff`；
- 组合逻辑使用 `always_comb`；
- 不在多个 always 块驱动同一寄存器；
- RAM 数组不全量复位；
- byte enable 写法固定；
- 乘除法 Backend 延迟写进接口契约；
- 不在关键路径堆巨大动态循环；
- 避免不受约束的跨时钟组合路径。

### 11.6 固定报告

每次 FPGA Release 自动产生：

```text
report_timing_summary
report_utilization -hierarchical
report_clock_interaction
report_cdc
report_drc
report_power（可选）
report_blackbox
```

当前 Tcl 已有 sanity report 思路，应继续保留并按新层次更新实例名。

---

## 12. Package、参数和配置怎样管理

### 12.1 Package 的职责

推荐：

```text
cpu_config_pkg.sv
    CPU 队列深度、Cache 参数、功能开关

cpu_types_pkg.sv
    uop、completion、branch 等 CPU 内部类型

hxi_pkg.sv
    HXI 宽度、错误和操作类型

memory_map_pkg.sv
    Base/Mask、地址属性

soc_config_pkg.sv
    Master/Slave 数量、外设使能
```

### 12.2 不要重复计算派生参数

例如：

```text
DCACHE_LINE_BYTES = 16
DCACHE_WORDS_PER_LINE = 4
```

应在一个 Owner Package 中定义或通过一个函数计算。

不能在三个文件里各写一份：

```systemverilog
localparam WORDS = LINE_BYTES / 4;
```

然后未来只修改其中两份。

### 12.3 少用全局宏

不推荐：

```text
`CPU_WIDTH
`CACHE_SIZE
`SOC_UART_ADDR
```

宏没有命名空间，容易因编译顺序产生隐式依赖。

优先：

```systemverilog
cpu_config_pkg::XLEN
memory_map_pkg::UART_BASE
```

### 12.4 参数不要从顶层一路重复传到底层

只让真正可配置且由上层决定的值成为 parameter。

如果一个参数改变公开端口宽度，必须由同一层统一拥有，不能在不同模块中用不同默认值。

### 12.5 编译期配置与运行时配置分开

```text
编译期：
    Cache 容量、Master 数量、FIFO 深度

运行时：
    UART enable、IRQ mask、Timer compare
```

不要用 `ifdef` 代替软件寄存器，也不要为每个运行模式重新综合整套 SoC。

---

## 13. 时钟、复位和 CDC 的目录责任

### 13.1 统一内部复位风格

推荐内部统一：

```text
clk_i
rst_i    高有效、同步使用
```

板级异步复位先在每个时钟域经过：

```text
异步置位、同步释放
```

再送入功能模块。

如果保留低有效风格也可以，但整个 `rtl/` 必须统一，Wrapper 负责极性转换。

### 13.2 每个时钟域有自己的 Reset Synchronizer

```text
global reset
├─ reset_sync(cpu_clk)  → cpu_rst
└─ reset_sync(uart_clk) → uart_rst
```

不能把 `cpu_rst` 直接拿去复位另一个异步域。

### 13.3 CDC 模块集中

通用同步器集中放：

```text
rtl/common/sync/
```

使用者明确实例化：

```text
bit_sync
pulse_sync
async_fifo
```

不要每个外设都手写不同版本的两级同步器。

### 13.4 谁拥有 CDC

跨域信号的接收模块负责同步，系统文档负责列出全部 CDC 边界。

例如：

```text
UART RX pin → UART clock domain
virtual switch → CPU domain
Timer IRQ → CPU domain
```

多位数据不能逐位用两级同步器，应使用握手、Gray Code 或 Async FIFO。

---

## 14. 命名规范

推荐统一：

```text
模块：snake_case
    hxi_crossbar
    machine_timer

文件名与主模块名一致
    machine_timer.sv

输入：*_i
输出：*_o
双向：*_io

当前状态寄存器：*_q
下一状态：*_d
组合值：*_c

低有效：*_ni / *_no

时钟：clk_i
复位：rst_i 或 rst_ni

请求/响应：
    req_*
    rsp_*

中断：
    irq_*_i
    irq_*_o
```

Package：

```text
*_pkg.sv
```

顶层：

```text
cpu_sim_top
soc_sim_top
soc_core
fpga_top
```

不要继续混用：

```text
w_cpu_clk
cpu_clk
clk
i_clk
```

除非它们确实属于不同边界，并在 Wrapper 中转换。

---

## 15. 断言和 Debug 怎样组织

### 15.1 协议断言靠近协议 Owner

例如 HXI：

```text
valid && !ready 时 payload 保持
一笔请求只命中一个 Slave
一笔响应只返回一个 Master
单 outstanding 不重复接收
```

这些断言放在：

```text
hxi_assertions.sv
```

或由 `bind` 绑定，不要散在 C++ Harness。

### 15.2 功能 Debug 用稳定 Probe

CPU 建议统一：

```text
commit_valid
commit_pc
commit_inst
commit_wen
commit_rd
commit_wdata
commit_mem_addr
commit_trap
commit_cause
```

SoC 建议：

```text
last_hxi_request
last_hxi_response
mtime/mtimecmp/mtip
uart_tx_count
irq_pending
```

### 15.3 仿真和 FPGA 共用观测语义

Verilator 使用这些信号生成日志。

FPGA 需要时由 ILA 连接相同信号。

这样不会出现：

```text
仿真看到一种 Debug
FPGA 只能观察完全不同的内部节点
```

---

## 16. 构建目标建议

顶层 Makefile 只提供易记入口，实际工作交给脚本。

```text
make lint
make lint-core
make lint-soc

make sim-unit TEST=timer
make sim-unit TEST=hxi

make sim-isa TEST=rv32ui-p-add
make sim-isa-all

make sim-soc TEST=hello_uart
make sim-soc TEST=timer_irq
make sim-soc TEST=rtthread
make sim-soc TEST=coremark

make fpga-elab BOARD=kintex7_competition
make fpga-build BOARD=kintex7_competition PROFILE=rtthread
make fpga-report BOARD=kintex7_competition

make software PROFILE=rtthread
make image PROFILE=rtthread
make regression
```

结果统一放：

```text
build/result/<target>/<test>.json
build/log/<target>/<test>.log
```

当前 `run_verilator.py` 和 JSON 结果机制可以继续扩展，不必推翻。

---

## 17. 设计源、平台源和生成物的边界

| 内容 | 是否提交 Git | 位置 |
| --- | ---: | --- |
| CPU/SoC RTL | 是 | `rtl/` |
| Verilator Model/Harness | 是 | `tb/` |
| Filelist/flags/waiver | 是 | `sim/` |
| Vivado Tcl/XDC | 是 | `fpga/` |
| IP 参数创建 Tcl | 是 | `fpga/` |
| `.xci` | 视流程而定；若可由 Tcl 稳定重建可不提交 | `fpga/` |
| `.xpr/.runs/.cache` | 否 | `build/vivado/` |
| Verilator `obj_dir` | 否 | `build/verilator/` |
| ELF/map/dump | Release 可归档，普通构建不提交 | `build/software/` |
| HEX/COE/MEM | 输入 Profile 可提交，临时生成不提交 | `data/profiles/` 或 `build/` |
| 波形 | 否 | `build/wave/` |
| 测试期望值 | 是 | `sim/regression/` |

原则：

```text
能够从源码和脚本重建的中间产物，不作为唯一设计源。
```

---

## 18. 与三人分工的对应关系

结合 12 号文档：

### 队长

```text
rtl/cpu/**
rtl/soc/**
rtl/common/pkg/架构常量
docs/architecture/**
docs/interfaces/**
顶层 filelist
Release
```

### 队员 A

```text
rtl/bus/**
rtl/memory/**
rtl/peripheral/**
rtl/common/sync/**
tb/unit/hxi|memory|timer|uart|irq
fpga/**
外设寄存器文档
```

### 队员 B

```text
software/**
tb/tests/baremetal|rtthread|coremark
tb/cpp/系统 Harness
sim/regression/**
软件构建和镜像工具
系统结果日志
```

共享文件不应让三个人随意修改。

例如：

```text
memory_map_pkg.sv
```

由队长批准，A 提案硬件窗口，B 评审软件可用性。

---

## 19. 当前文件怎样迁移到新结构

### 19.1 CPU

```text
当前 rtl/core/pkg/
→ rtl/cpu/pkg/

当前 rtl/core/frontend/
→ rtl/cpu/frontend/

当前 rtl/core/memory/dcache*
→ rtl/cpu/cache/dcache/

当前 rtl/core/memory/store_buffer/load_queue/...
→ rtl/cpu/memory/

当前 rtl/core/commit/csr_file.sv
→ rtl/cpu/csr_trap/

当前 rtl/core/core_top.sv
→ rtl/cpu/cpu_core.sv

当前 rtl/core/myCPU.sv
→ rtl/cpu/cpu_subsystem.sv 或 cpu_soc_wrapper.sv
```

第一阶段可以只调整 filelist 和文档，不立即重命名模块。

### 19.2 SoC

```text
rtl/soc/student_top.sv
→ rtl/soc/soc_core.sv（逐步演进）

rtl/soc/SocMemBridge.sv
→ 被 rtl/bus/hxi + rtl/memory/adapter + peripheral 替代

rtl/soc/DramBramAdapter.sv
→ rtl/memory/adapter/

rtl/soc/counter.sv
→ rtl/peripheral/perf_ctrl/ 或 legacy_counter/

rtl/soc/uart.sv
→ 区分物理 UART 功能、APB UART 与 Host/Twin
```

### 19.3 板级

```text
rtl/soc/top.sv
→ fpga/boards/kintex7_competition/rtl/fpga_top.sv

rtl/soc/twin_controller.sv
→ fpga/boards/kintex7_competition/rtl/digital_twin_wrapper.sv

rtl/soc/display_seg.sv / seg7.sv
→ fpga/boards/.../rtl/ 或通用 GPIO 显示外设
```

如果数码管确实是 SoC 软件可见外设，其寄存器部分放 `rtl/peripheral/`，物理扫描部分放板级目录。

### 19.4 仿真模型

```text
rtl/ip/*.sv
→ tb/models/xilinx_compat/
```

Verilator filelist 选择它们；FPGA filelist禁止选择它们。

---

## 20. 不要一次完成“大迁移”

推荐分六步。

### 阶段 0：建立基线

- 记录当前 CPU/SoC Verilator 测试；
- 记录 FPGA 综合命令；
- 保存结果 JSON；
- 确认工作树状态；
- 不改变行为。

### 阶段 1：只整理 Filelist

- 建立 `pkg/common/cpu/soc` 层次；
- 源文件暂时不移动；
- Verilator 和 Vivado 读取同一设计清单；
- 增加 filelist 检查。

### 阶段 2：分离三个 Top

- `cpu_sim_top`；
- `soc_sim_top`；
- `fpga_top`；
- 保持功能不变。

### 阶段 3：移动仿真模型和板级文件

- `rtl/ip` → `tb/models`；
- FPGA Top → `fpga/boards`；
- Tcl/XDC 保持可重建。

### 阶段 4：建立 HXI/Memory/Peripheral 目录

- 先移动或包裹现有模块；
- 再逐步用 Crossbar、APB、Timer 替换 `SocMemBridge`；
- 每步保持 Smoke Test。

### 阶段 5：CPU 内部整理

- Cache 与 LSU 分开；
- CSR/Trap 分开；
- 移除改变端口的 `VERILATOR_TB`；
- 保持 ISA 回归。

### 阶段 6：清理 Legacy

只有：

- 新结构测试全通过；
- Vivado 工程能重建；
- FPGA 能启动；

以后才删除旧路径和兼容 Wrapper。

---

## 21. 每次结构调整必须跑哪些测试

### 只移动文件

```text
Verilator CPU build
Verilator SoC build
Vivado elaboration
```

### 修改 CPU 目录或接口

```text
ISA directed
异常/CSR
CPU build
SoC build
```

### 修改 HXI/Memory

```text
HXI unit
BRAM adapter unit
CPU load/store
SoC smoke
非法地址
```

### 修改 Timer/IRQ

```text
Timer unit
CPU IRQ injection
baremetal timer IRQ
RT-Thread delay
```

### 修改 FPGA Backend

```text
Vivado elaboration
synthesis
blackbox
utilization
timing
上板 UART smoke
```

---

## 22. 需要冻结的架构决策

目录可以先规划，但以下问题仍要在编码前明确。

### 必须尽快决定

1. I-side 是否第一版就改为 HXI；
2. Code BRAM 和 Data BRAM 容量；
3. `.rodata` 放哪里；
4. Native Memory Port 读延迟；
5. 默认采用 Generic RAM 还是 Xilinx IP Backend；
6. CPU 和 UART 是否保留两个时钟域；
7. Console UART 与 digital twin 是否共用物理串口；
8. Timer 是 HXI Slave 还是 APB Slave；
9. 第一版外部 IRQ 直连还是简化控制器；
10. Debug Port 是稳定端口还是独立 Trace Wrapper。

### 本文推荐的暂定答案

```text
I-side：
    先保留当前专用 IROM 接口，设计 HXI-I 边界后再切换

Memory：
    Code/Data 独立，D-side 至少能读取 .rodata 所在区域

Backend：
    Verilator 用行为 Model
    FPGA 用 Xilinx Wrapper
    二者遵守同一 Native Memory Port

Clock：
    CPU/SoC 尽量一个主域
    digital twin/UART 若独立则显式 CDC

Timer：
    独立 HXI Slave

IRQ：
    Timer 直连
    UART external IRQ 先行
    多外设后再加简化控制器

Debug：
    稳定 Commit Trace 接口，不用宏改变 CPU 端口
```

这些属于暂定架构，不应在没有评审时分散硬编码到各模块。

---

## 23. 最终验收标准

新的项目结构完成后应满足：

### 清晰性

- [ ] 每个目录只有一种主要职责；
- [ ] CPU 不依赖 FPGA IP；
- [ ] SoC 功能与板级管脚分离；
- [ ] 仿真模型不在综合 filelist；
- [ ] 生成物全部进入 `build/`；
- [ ] 模块 Owner 与目录一致。

### Verilator

- [ ] CPU 和 SoC 有独立 Top；
- [ ] 显式 `--top-module`；
- [ ] 独立 `--Mdir`；
- [ ] Filelist 无重复和缺失；
- [ ] 普通回归不生成波形；
- [ ] 模型支持可配置延迟/背压；
- [ ] 不依赖易碎的深层层次路径；
- [ ] Warning/Waiver 可审查。

### FPGA

- [ ] Vivado 工程可由 Tcl 重建；
- [ ] XDC 按板卡和职责组织；
- [ ] IP 参数由 Tcl 或受控配置生成；
- [ ] Vendor IP 只通过 Wrapper 接入；
- [ ] RAM 能正确推断或映射 BRAM；
- [ ] 时钟/复位/CDC 边界明确；
- [ ] 固定生成时序、资源、CDC、DRC 和 Blackbox 报告；
- [ ] 同一软件镜像能在 SoC 仿真和 FPGA 使用。

### 维护性

- [ ] Memory Map 只有一个权威来源；
- [ ] Package 编译顺序明确；
- [ ] 参数 Owner 明确；
- [ ] 编译宏不改变核心功能；
- [ ] 结构迁移有阶段性兼容方案；
- [ ] 主分支始终保留可运行 Smoke Test。

---

## 24. 最重要的判断

“Verilator 友好”和“FPGA 友好”不是写两套 RTL。

真正合适的结构是：

```text
同一份功能 RTL
    │
    ├─ 接 Verilator Model → 快速仿真和回归
    │
    └─ 接 Xilinx Backend → 综合、实现和上板
```

应该发生变化的是：

- 最外层 Top；
- 时钟和复位来源；
- 存储/算术 Backend；
- 外部 Host/引脚模型；
- 构建脚本；

不应发生变化的是：

- CPU 行为；
- HXI 协议；
- Cache；
- Timer/UART 寄存器；
- Memory Map；
- 中断语义；
- 软件看到的系统。

只要把这个边界守住，Verilator 仿真结果才真正能够代表 FPGA 上的设计。

---

## 25. 参考资料

- [Verilator 官方文档：Verilating 与 Top 选择](https://verilator.org/guide/latest/verilating.html)
- [Verilator 官方参数：`--top-module`、`--Mdir`、Trace](https://verilator.org/guide/latest/exe_verilator.html)
- [AMD Vivado UG892：Project Mode Tcl Flow](https://docs.amd.com/r/2022.2-English/ug892-vivado-design-flows-overview/Using-Project-Mode-Tcl-Commands)
- [AMD Vivado UG895：使用 Tcl 添加设计源、约束与仿真源](https://docs.amd.com/r/2023.2-English/ug895-vivado-system-level-design-entry/Tcl-Commands-for-Adding-Design-Sources-Constraints-Files-and-Simulation-Sources)
- [AMD Vivado UG896：IP Tcl Flow](https://docs.amd.com/r/2021.2-English/ug896-vivado-ip/Using-IP-Tcl-Commands-In-Design-Flows)
- [11：SoC 各组成部分的职责、接口与设计理由](./11_soc_components_roles_interfaces_and_design.md)
- [12：三人团队的职责边界、交付标准与协作方式](./12_three_person_team_roles_boundaries_and_delivery.md)

