# SoC 各组成部分：职责、接口与设计理由

> 本文面向有计算机组成原理和 CPU RTL 基础，但不熟悉 SoC 集成、外设、片上总线和操作系统硬件需求的读者。  
> 文中以当前 `superScalar` 项目的 RV32 CPU、HXI 互联、FPGA BRAM、RT-Thread 移植目标为背景；同时保留通用 SoC 的理解方式。  
> 本文讲的是“模块应该做什么、怎样连接”，不是一份需要立即照抄的 RTL 端口定义。

---

## 0. SoC 究竟是什么

SoC 是：

```text
System on Chip
片上系统
```

你已经会设计 CPU 内部的数据通路和控制逻辑。SoC 工作是在 CPU 周围补上一整台计算机所需的其他部分：

```text
一颗能执行指令的 CPU
    + 存放程序和数据的存储器
    + 让各模块通信的片上互联
    + 与外界通信的 UART/GPIO 等外设
    + 提供操作系统节拍的 Timer
    + 收集和管理事件的中断系统
    + 产生时钟和复位的基础设施
    + FPGA 板级引脚、PLL 和 BRAM IP
    = 一个能够独立运行软件的 SoC
```

CPU 解决的是：

```text
“怎样正确、高效地执行一条条指令？”
```

SoC 解决的是：

```text
“指令和数据放在哪里？”
“CPU 访问地址时，应该送到哪个硬件？”
“串口收到字符后，怎样通知 CPU？”
“操作系统每隔 1 ms 怎样得到一次 Tick？”
“程序怎样从 FPGA BRAM 启动？”
“所有模块怎样在同一个时钟、复位和地址空间下工作？”
```

这就是从“CPU 核设计”走向“完整计算机系统设计”的变化。

---

## 1. 先看一张完整的 SoC 结构图

推荐你把整个目标系统理解成下面几层：

```text
FPGA 板级顶层 fpga_top
├─ 差分时钟输入
├─ PLL / Clock Wizard
├─ 外部复位
├─ UART 引脚
├─ GPIO 引脚
└─ soc_top
   │
   ├─ cpu_subsystem
   │  ├─ CPU core
   │  ├─ CSR / Trap / IRQ
   │  ├─ I-cache（可选）
   │  ├─ D-cache
   │  ├─ Store Buffer
   │  ├─ HXI-I Master
   │  └─ HXI-D Master
   │
   ├─ hxi_crossbar
   │  ├─ 地址译码
   │  ├─ 请求路由
   │  ├─ 仲裁
   │  ├─ 响应路由
   │  └─ Default Error Slave
   │
   ├─ memory_subsystem
   │  ├─ Code BRAM Adapter → Code BRAM
   │  └─ Data BRAM Adapter → Data BRAM
   │
   ├─ machine_timer / CLINT-MTIMER
   │
   ├─ interrupt_controller
   │
   └─ hxi_to_apb_bridge
      └─ apb_peripheral_subsystem
         ├─ UART
         ├─ GPIO
         ├─ 性能控制寄存器
         └─ 后续 SPI/PWM 等外设
```

这里有两种完全不同的“连接”：

### 数据和配置连接

```text
CPU → HXI → 存储器/外设寄存器
```

CPU 通过 Load/Store 读写其他模块。

### 事件连接

```text
Timer/UART/GPIO → 中断线 → CPU
```

外设主动告诉 CPU“发生了事情”。

学习 SoC 时，首先要避免把这两类连接混在一起。

---

## 2. 理解接口前，先认识几个常用词

### 2.1 模块

模块就是一块具有明确职责的硬件。

例如：

```text
CPU core        执行指令
UART            收发串口字节
Timer           到点产生中断
Address Decoder 根据地址选择目标设备
```

一个好的模块不应该同时承担大量互不相关的职责。

### 2.2 接口

接口是两个模块之间约定好的通信方式。

它不只是端口名称，还包括：

- 哪一方发起；
- 哪一方响应；
- 信号在哪个周期有效；
- 能不能等待；
- 出错怎样表示；
- 复位时是什么状态；
- 一次请求何时算完成。

本文把接口分成三类。

#### 数据接口

传递真正的内容：

```text
地址
写数据
读数据
指令
UART 字节
GPIO 电平
```

#### 控制接口

说明数据何时有效、是否被接收：

```text
valid
ready
write
byte enable
enable
error
```

#### 事件接口

不传输大块数据，只通知发生了事件：

```text
irq_timer
irq_external
uart_rx_irq
```

### 2.3 主设备 Master

Master 是主动发起访问的一方。

例如：

```text
CPU 取指接口
CPU 数据接口
DMA
调试下载器
```

它们会提出：

```text
我要读地址 A
我要把数据 D 写到地址 B
```

### 2.4 从设备 Slave

Slave 是接受请求并返回结果的一方。

例如：

```text
BRAM
UART
GPIO
Timer
中断控制器
```

“主从”只描述一次总线访问中的主动方和被动方，不表示模块的重要程度。

### 2.5 Transaction

Transaction 可以翻译成“事务”或“一笔访问”。

例如一次 32 位读取：

```text
请求：读取地址 0x8010_0100
响应：返回 0x1234_5678
```

请求和响应合起来才是一笔完整事务。

### 2.6 MMIO

MMIO 是：

```text
Memory-Mapped I/O
内存映射 I/O
```

它把外设寄存器放进 CPU 的统一地址空间。

例如：

```c
*(volatile unsigned int *)UART_TX_ADDR = 'A';
```

CPU 执行的是普通 Store，但地址译码发现它不属于 RAM，而属于 UART，于是这一笔写访问变成了“发送字符 A”。

### 2.7 寄存器映射

外设内部有一组可由 CPU 读写的控制和状态寄存器，例如：

```text
UART_BASE + 0x00  DATA
UART_BASE + 0x04  STATUS
UART_BASE + 0x08  CTRL
UART_BASE + 0x0C  BAUD
```

寄存器映射必须说明：

- 地址偏移；
- 位宽；
- 读写属性；
- 复位值；
- 每一位的含义；
- 写入是否产生副作用；
- 中断怎样清除。

### 2.8 中断 IRQ

IRQ 是 Interrupt Request，中断请求。

它表示：

```text
某个异步事件已经发生，希望 CPU 尽快处理
```

CPU 不会因为看到 IRQ 就在任意组合逻辑中突然跳走，而是在满足使能条件后，于精确指令边界进入 Trap。

### 2.9 Backpressure

Backpressure 是“接收方暂时接不下，要求发送方等待”。

在 `valid/ready` 接口中：

```text
valid = 1：发送方提供了有效内容
ready = 1：接收方现在可以接收
```

只有：

```text
valid && ready
```

同时为 1 的时钟边沿，传输才真正发生。

如果：

```text
valid = 1
ready = 0
```

发送方必须保持地址、数据和控制字段不变。这就是接口协议的一部分。

---

## 3. FPGA 板级顶层：把芯片内部接到真实引脚

### 3.1 它负责什么

板级顶层通常负责：

- 接收 FPGA 板上的差分时钟；
- 实例化 PLL/Clock Wizard；
- 产生 SoC 所需的时钟；
- 处理外部复位；
- 连接 UART RX/TX 引脚；
- 连接 LED、按键、拨码开关等引脚；
- 实例化真正的 `soc_top`。

当前工程中的 `rtl/soc/top.sv` 就属于这一层。

### 3.2 它的外部接口

典型接口：

```text
输入：
    i_sys_clk_p / i_sys_clk_n   板级差分时钟
    i_reset_n                   板级复位
    i_uart_rx                   串口接收
    i_gpio[...]                 外部输入

输出：
    o_uart_tx                   串口发送
    o_gpio[...]                 外部输出
    LED/SEG                     板上显示
```

这些端口最终会在 XDC 约束文件中绑定到 FPGA 的实际管脚。

### 3.3 它为什么要与 `soc_top` 分开

因为板级接口依赖具体 FPGA 和开发板：

```text
差分还是单端时钟
晶振频率
PLL IP
UART 管脚
复位按键极性
LED 数量
```

而 SoC 内部架构应尽量与板卡无关。

分开以后：

```text
同一个 soc_top
├─ 可以接 Kintex-7 板级顶层
├─ 可以接另一块 FPGA 板
└─ 可以接 Verilator 仿真顶层
```

### 3.4 当前工程需要注意什么

当前 `top.sv` 中 UART 主要服务于 `twin_controller`，不是 CPU 可通过 MMIO 直接控制的 RT-Thread Console UART。

因此未来可能有两种方案：

1. 保留 digital-twin UART，再增加一个 CPU MMIO UART；
2. 重新定义复用协议，让同一个物理 UART 在调试协议和 RT-Thread Console 间可靠切换。

第一种更容易验证，但要确认开发板是否有足够的物理串口资源。

---

## 4. SoC 顶层：完成系统集成

### 4.1 它负责什么

`soc_top` 的任务不是执行具体算法，而是：

- 实例化 CPU 子系统；
- 实例化互联；
- 实例化存储器和外设；
- 连接中断；
- 分发时钟和复位；
- 汇总 SoC 对外接口；
- 明确整个地址空间。

可以把它理解为芯片内部的“总装图”。

### 4.2 它的典型接口

```text
输入：
    soc_clk
    soc_rst
    uart_rx
    gpio_in
    debug inputs

输出：
    uart_tx
    gpio_out
    debug/status outputs
```

如果 PLL 放在板级顶层，`soc_top` 不需要知道差分时钟和具体 PLL 型号，只接收已经处理好的 `soc_clk`。

### 4.3 为什么不应该把所有 RTL 都写在 SoC 顶层

如果顶层内部直接写：

- 地址译码；
- UART 状态机；
- Timer 计数；
- BRAM 控制；
- 中断优先级；
- 复位同步；

那么顶层会变成难以验证的巨型模块。

正确做法是让顶层只表达：

```text
有哪些模块
它们怎样连接
参数是什么
```

具体行为放到各自模块中。

---

## 5. CPU Core：只负责“执行指令”

### 5.1 CPU Core 的职责

CPU 核内部负责：

- 取指；
- 译码；
- 寄存器重命名或普通寄存器依赖处理；
- 执行整数、乘除法、分支；
- Load/Store 请求生成；
- 异常检测；
- CSR；
- 精确提交；
- 中断进入和 `mret` 返回；
- Cache（常放在 CPU 子系统内部）。

CPU 不应该知道：

- FPGA 上 UART 具体连到哪个管脚；
- GPIO 有多少个 LED；
- BRAM 使用哪一种 Xilinx IP；
- 某个外设的 APB 内部状态机；
- 链接脚本把 `.data` 放在哪一块物理 BRAM。

### 5.2 CPU Core 面向 SoC 的接口

一个通用 CPU 子系统至少需要四类接口。

#### 指令访问接口

```text
输出：
    i_req_valid
    i_req_addr

输入：
    i_req_ready
    i_rsp_valid
    i_rsp_data
    i_rsp_error
```

含义是：

```text
CPU 请求读取 PC 地址处的指令
存储系统稍后返回指令或错误
```

当前项目仍使用较直接的 `irom_addr/irom_data/irom_ena` 接口。它适合专用同步 IROM，但如果未来 I-side 也接 HXI Crossbar，最好规范成请求/响应接口。

#### 数据访问接口

当前项目已有类似：

```text
dmem_req_valid
dmem_req_ready
dmem_req_write
dmem_req_addr
dmem_req_wdata
dmem_req_wstrb
dmem_req_uncached

dmem_resp_valid
dmem_resp_rdata
```

它表达：

```text
CPU 希望读/写某个地址
SoC 是否接收
读取结果何时返回
这是普通内存还是不可缓存设备访问
```

未来还建议补：

```text
dmem_resp_error
```

这样访问非法地址时，CPU 可以产生 Load/Store Access Fault，而不是永远等待或默默返回 0。

#### 中断接口

```text
irq_software_i
irq_timer_i
irq_external_i
```

它们分别对应：

- 软件中断；
-机器定时器中断；
- UART/GPIO 等外部中断汇总。

#### 调试接口

可选：

```text
提交 PC
提交指令
写回寄存器
访存地址
Trap cause
性能计数
```

这些接口用于仿真、差分测试、ILA 和性能分析，通常不属于软件可见功能。

### 5.3 为什么 CPU 和 SoC 要划清边界

如果 CPU 直接实例化 UART、GPIO 和板级 BRAM：

- CPU 无法脱离这块板独立测试；
- 换地址会修改 CPU；
- 换 UART 会修改流水线工程；
- Cache 难以区分普通内存与 MMIO；
- SoC 扩展 DMA 或 Debug Master 会很困难。

合理边界是：

```text
CPU 只产生规范化的存储访问和中断响应
SoC 决定每个地址对应什么设备
```

---

## 6. CPU Wrapper 与 CPU Subsystem

### 6.1 Wrapper 是什么

Wrapper 是“适配外壳”。

它本身通常不实现复杂 CPU 算法，而是完成：

- 复位极性转换；
- 端口命名转换；
- 总线协议适配；
- Cacheable/Uncached 属性生成；
- 中断线整理；
- 调试接口裁剪；
- 参数固定。

例如：

```text
core 内部使用 active-low reset
SoC 使用 active-high reset
```

Wrapper 可以在边界统一转换，而不是让整个工程混用复位极性。

### 6.2 CPU Subsystem 比 Core 多了什么

推荐层次：

```text
cpu_subsystem
├─ core_pipeline
├─ CSR/Trap
├─ I-cache / Fetch Queue
├─ D-cache
├─ Store Buffer
├─ I-side bus adapter
└─ D-side bus adapter
```

从软件角度，它们都属于“处理器”；从 RTL 组织角度，Core 和 Cache/Bus Adapter 可以分层。

### 6.3 Cache 为什么放在 CPU 与总线之间

```text
CPU Pipeline
    ↓
Cache
    ↓
HXI
    ↓
BRAM/DDR
```

Cache 截获大多数普通内存访问：

- 命中时直接返回；
- 缺失时通过 HXI 请求主存；
- MMIO 地址强制绕过 Cache。

它不应该放在 UART 或 Timer 后面，也不应该缓存外设寄存器。

---

## 7. 片上总线/互联：SoC 的通信骨架

### 7.1 它负责什么

互联负责把多个 Master 与多个 Slave 连接起来：

```text
Master
├─ CPU I-side
├─ CPU D-side
├─ DMA（以后）
└─ Debug Loader（以后）

Slave
├─ Code BRAM
├─ Data BRAM
├─ Timer
├─ Interrupt Controller
├─ APB Bridge
└─ Default Error Slave
```

它的核心职责：

1. 根据地址选择目标 Slave；
2. 把请求送到正确目标；
3. 多个 Master 争用同一 Slave 时进行仲裁；
4. 记录响应应该返回哪个 Master；
5. 传递等待和错误；
6. 保证协议不会丢请求、重复请求或串错响应。

### 7.2 总线为什么不是“一大捆公共导线”

当只有一个 CPU 和一个 RAM 时，可以直接连接。

但加入多个设备后会遇到：

```text
CPU-I 和 CPU-D 同时访问怎么办？
两个 Master 同时访问 RAM 怎么办？
UART 返回的数据怎样避免送给 IFU？
某个 Slave 等待十拍时，其他路径能否继续？
非法地址怎样结束访问？
```

所以 SoC 互联实际上包含译码、路由、状态记录和仲裁逻辑。

### 7.3 HXI 是什么

当前推荐的 HXI 是项目自定义的片上请求/响应协议，不是 AMBA 标准。

可采用：

```text
请求通道：
    req_valid
    req_ready
    req_addr
    req_write
    req_wdata
    req_wstrb
    req_size
    req_instr

响应通道：
    rsp_valid
    rsp_ready
    rsp_rdata
    rsp_err
```

它的优点是：

- 与当前 CPU 数据接口接近；
- 字段较少；
- 适合第一版单 outstanding；
- 容易针对 Cache、BRAM 和 MMIO 定义规则；
- 多主多从 Crossbar 可逐步实现。

它的代价是：

- 协议必须由你们自己写完整；
- 没有现成标准 IP 保证兼容；
- 验证责任在自己；
- 不能把“看起来像 AXI-Lite”称为“兼容 AXI-Lite”。

### 7.4 AMBA、AHB、AXI、APB 是什么

AMBA 是 Arm 定义的一组片上互联协议家族。

简单定位：

| 协议 | 常见用途 | 特点 |
| --- | --- | --- |
| AHB-Lite | MCU CPU、SRAM、DMA 主干 | 地址/数据相位流水，接口相对适中 |
| AXI4 | 高性能、多 outstanding、DDR | 五通道、支持 burst，复杂 |
| AXI4-Lite | 寄存器控制访问 | AXI 风格但不支持 burst |
| APB | UART/GPIO/Timer 等低速寄存器外设 | 非流水、状态机简单 |

HXI 不需要为了“更专业”而强行改成 AXI。对比赛项目，真正重要的是：

```text
协议语义明确
等待正确
响应不串线
错误可返回
验证足够
```

### 7.5 为什么常用“主干互联 + APB 子系统”

UART、GPIO 等外设带宽很低，如果每个都直接占用 Crossbar 一个复杂 Slave 端口，会让主干不断膨胀。

因此常见结构是：

```text
高速/主干互联
├─ RAM
├─ ROM
├─ Timer
└─ HXI-to-APB Bridge
   └─ APB Interconnect
      ├─ UART
      ├─ GPIO
      └─ 其他低速外设
```

Arm 的官方 AMBA 资料也把 APB定位为低带宽控制和外设寄存器访问，总线不做复杂流水，有利于降低外设实现复杂度。

---

## 8. 地址译码器：决定“这个地址属于谁”

### 8.1 它负责什么

CPU 发出一个 32 位地址，译码器判断命中哪个区域：

```text
0x8000_xxxx → Code BRAM
0x8010_xxxx → Data BRAM
0x8020_xxxx → 外设窗口
其他地址    → Default Error Slave
```

以上只是表达当前工程的地址组织方式；最终范围必须在正式 Memory Map 中冻结。

### 8.2 它的接口

输入：

```text
req_addr
可能还有 req_instr、req_write、权限属性
```

输出：

```text
select_code_bram
select_data_bram
select_apb_bridge
select_timer
select_irq_ctrl
select_default
```

### 8.3 为什么必须独立规划 Memory Map

Memory Map 同时影响：

- HXI 地址译码；
- CPU Cacheable/Uncached 判断；
- 链接脚本；
- BSP 寄存器宏；
- 仿真模型；
- ELF/HEX/COE 镜像生成；
- 软件异常处理；
- 技术文档。

如果不同模块各自硬编码地址，很容易出现：

```text
RTL 认为 UART 在 A
BSP 认为 UART 在 B
链接脚本又把 RAM 放到 C
```

建议集中维护：

```text
rtl/memory/memory_map_pkg.sv
bsp/soc_memory_map.h
docs/memory_map.md
```

### 8.4 地址属性

地址不只表示“访问谁”，还应有属性：

```text
Code BRAM：
    readable
    executable
    cacheable

Data BRAM：
    readable/writable
    cacheable
    是否 executable 由设计决定

MMIO：
    readable/writable 视寄存器而定
    non-cacheable
    non-executable
    不可随意预取或乱序
```

---

## 9. 仲裁器：解决多个 Master 同时访问

### 9.1 什么情况下需要仲裁

例如同一周期：

```text
IFU 想读 Code BRAM
LSU 也想从 Code BRAM 读取 .rodata
```

如果 Code BRAM Slave 每次只能服务一笔请求，就必须选择一个。

### 9.2 仲裁器的功能

输入：

```text
来自 M0、M1、M2 的请求
```

输出：

```text
当前谁获准访问 Slave
其他请求的 ready = 0
```

常见策略：

#### 固定优先级

```text
M0 > M1 > M2
```

简单，但低优先级可能长期得不到服务，称为 starvation（饥饿）。

#### Round-Robin

每次服务后轮换优先权。

优点是公平，适合当前多主互联的第一版。

### 9.3 为什么要记录 Owner

BRAM 可能在请求后一个或多个周期才返回数据。

请求发生时：

```text
M1 → Data BRAM
```

响应回来时，当前总线地址可能早已变化。所以互联必须保存：

```text
这个 Slave 当前事务属于 M1
```

再把响应送回 M1。

不能根据响应时刻的地址重新猜测接收者。

---

## 10. Default Error Slave：让非法访问能够结束

### 10.1 它负责什么

如果 CPU 访问一个没有设备的地址：

```text
0xDEAD_0000
```

互联必须给出明确响应：

```text
rsp_valid = 1
rsp_err   = 1
```

CPU 再把它转换为：

- Instruction Access Fault；
- Load Access Fault；
- Store/AMO Access Fault。

### 10.2 为什么不能简单返回 0

返回 0 会掩盖软件错误：

```text
UART 地址写错了
软件读到 0
继续运行
最后在完全不相关的位置崩溃
```

### 10.3 为什么不能永远不响应

CPU 会永久等待，整个系统死锁，而且很难定位原因。

Default Slave 的价值是：

```text
任何地址访问最终都必须“正常完成”或“明确报错”
```

---

## 11. Code BRAM / I-ROM：存放程序映像

### 11.1 它负责什么

Code BRAM 存放：

- `.text` 指令；
- 启动代码；
- Trap 入口；
- 可能还包括 `.rodata`；
- FPGA 上电时预初始化的程序镜像。

CPU 复位后从固定 Reset Vector 开始取指，这个地址必须落在 Code BRAM 或 Boot ROM 中。

### 11.2 它的接口

BRAM 原生接口常见：

```text
clk
enable
address
write_enable
write_data
read_data
```

但 HXI 使用请求/响应。

因此需要：

```text
HXI
    ↓
hxi_bram_slave / BRAM Adapter
    ↓
Xilinx BRAM 原生接口
```

Adapter 负责把总线访问翻译成 BRAM 时序。

### 11.3 为什么需要 Adapter

BRAM 通常是同步读取：

```text
周期 N：给地址并使能
周期 N+1：读数据有效
```

HXI 则要求明确：

```text
何时接收请求
何时返回响应
```

Adapter 用状态机或流水寄存器把两种时序连接起来。

### 11.4 `.rodata` 为什么会影响连接

字符串常量和 `const` 数据虽然可能与代码一起放在 Code BRAM，但 C 程序读取它们时执行的是 Load，走 LSU/D-side。

因此至少要满足一种方案：

1. D-side 可以只读 Code BRAM；
2. `.rodata` 放进 Data BRAM；
3. 采用统一存储器。

如果只有 IFU 能访问 Code BRAM，而链接脚本把 `.rodata` 放在那里，程序会在打印字符串时失败。

### 11.5 FPGA BRAM 的意义

BRAM 是 FPGA 内部专用存储资源，不需要用大量触发器拼出 RAM。

可以通过：

- Vivado Block Memory Generator IP；
- 合适的 RTL 数组让工具推断 BRAM；

实现 ROM、单口 RAM、简单双口 RAM或真正双口 RAM。

当前阶段应优先统一“软件可见行为”，再决定最终使用 IP 还是推断方式：

```text
地址宽度
数据宽度
读延迟
写掩码
初始化文件
双口冲突规则
```

---

## 12. Data BRAM / D-RAM：保存运行时状态

### 12.1 它负责什么

Data BRAM 通常存放：

- `.data`；
- `.bss`；
- 堆；
- 主线程栈；
- 中断栈；
- RT-Thread 各线程栈；
- 内核对象；
- CoreMark 工作数组。

### 12.2 它的接口

数据访问需要支持：

```text
32 位读取
32 位写入
byte 写入
halfword 写入
字节写掩码 wstrb[3:0]
```

例如地址低两位和 `wstrb` 共同决定写哪个字节。

### 12.3 为什么 Code BRAM 和 Data BRAM 可以分开

分开能够让：

```text
IFU 读 Code BRAM
LSU 同时访问 Data BRAM
```

两条路径并行，减少结构冲突。

这接近 Harvard 风格：

```text
指令存储路径与数据存储路径分离
```

但软件仍然看到统一的 32 位地址空间。

### 12.4 为什么也有人用统一 RAM

统一 RAM 的优点：

- 链接脚本简单；
- 代码、只读数据和普通数据都能从同一存储器访问；
- 动态加载和执行代码更方便；
- 小型系统更容易 bring-up。

代价：

- IFU 和 LSU 会争用端口；
- 需要双口 RAM 或总线仲裁；
- 性能可能受影响。

当前项目已有分离 IROM/DRAM 基础，继续使用分离 BRAM是合理的，但必须解决 `.rodata` 和 D-side 访问范围。

---

## 13. BRAM Controller/Adapter：把存储 IP 变成总线设备

### 13.1 它负责什么

BRAM 本身不理解：

```text
req_valid
rsp_ready
总线错误
非对齐访问
地址范围
```

Adapter 负责：

- 锁存请求；
- 检查对齐和范围；
- 计算 BRAM word address；
- 产生 `enable` 和 byte write enable；
- 等待同步读数据；
- 返回响应；
- 在非法访问时返回 error。

### 13.2 为什么不让 Crossbar 直接连 Xilinx IP

如果 Crossbar 直接暴露 Xilinx 专有端口：

- 仿真行为模型和 FPGA IP 不容易替换；
- 每种 Memory Slave 都要在互联里写特殊逻辑；
- BRAM 读延迟变化会影响 Crossbar；
- 互联失去统一接口。

Adapter 隔离了：

```text
HXI 协议
    与
FPGA BRAM 实现细节
```

---

## 14. HXI-to-APB Bridge：连接主干与低速外设

### 14.1 Bridge 是什么

Bridge 是协议转换器。

上游看它：

```text
像一个 HXI Slave
```

下游看它：

```text
像一个 APB Master
```

### 14.2 它负责什么

一次流程：

```text
接收 HXI 请求
    ↓
锁存地址/读写方向/写数据/写掩码
    ↓
发起 APB SETUP
    ↓
发起 APB ACCESS
    ↓
等待 PREADY
    ↓
收集 PRDATA/PSLVERR
    ↓
生成 HXI 响应
```

### 14.3 为什么不让 CPU 直接说 APB

CPU 还需要高效访问 BRAM、Cache refill 和可能的 DMA。APB 不适合做整个系统的主干。

所以分层：

```text
HXI：主干通信
APB：低速寄存器外设
Bridge：连接两者
```

---

## 15. APB Interconnect：低速外设内部的地址分发

### 15.1 它负责什么

APB Bridge 后面仍可能有多个设备：

```text
UART
GPIO
性能控制寄存器
SPI
```

APB Interconnect 根据低位地址产生：

```text
PSEL_UART
PSEL_GPIO
PSEL_SPI
```

并把选中设备的：

```text
PRDATA
PREADY
PSLVERR
```

返回 Bridge。

### 15.2 为什么 APB 外设简单

外设通常只需实现：

```text
PSEL
PENABLE
PWRITE
PADDR
PWDATA
PSTRB
PRDATA
PREADY
PSLVERR
```

不需要处理复杂 outstanding、乱序响应和 Cache Line burst。

---

## 16. UART：把字节送到电脑终端

### 16.1 它负责什么

UART 完成串行字节收发：

```text
CPU 写 TXDATA
    ↓
UART 按波特率逐位发送到 TX 引脚

RX 引脚收到串行位流
    ↓
UART 恢复成一个字节
    ↓
CPU 读取 RXDATA
```

RT-Thread 的 FinSH/MSH 命令行依赖 Console UART。

### 16.2 UART 的两侧接口

#### SoC 总线侧

```text
DATA
STATUS
CTRL
BAUD
IRQ_ENABLE
IRQ_STATUS
```

#### 芯片引脚侧

```text
uart_rx
uart_tx
```

### 16.3 为什么需要 FIFO

CPU 和串口的速度差很多。

例如 CPU 50 MHz，而 UART 115200 baud。一个串口字节需要许多 CPU 周期。

FIFO 用来缓冲：

```text
TX FIFO：CPU 可以先写入几个字节，UART 慢慢发送
RX FIFO：UART 收到字符后先保存，等待 CPU 读取
```

第一版可以只有 1 字节缓冲，但 FinSH 连续输入时更容易丢字符。建议至少规划小型 RX/TX FIFO。

### 16.4 UART 中断

典型中断源：

```text
RX FIFO 非空
TX FIFO 有空间
接收错误
```

UART IRQ 通常进入外部中断控制器，再以 `irq_external_i` 通知 CPU。

最简 bring-up 可以先轮询 UART；RT-Thread Console 稳定后再启用接收中断。但最终交互式 FinSH 更适合中断接收。

---

## 17. GPIO：软件可控制或观察的引脚

### 17.1 它负责什么

GPIO 是 General-Purpose Input/Output，通用输入输出。

可能连接：

- LED；
- 按键；
- 拨码开关；
- 外部控制信号；
- 简单传感器中断。

### 17.2 常见寄存器

```text
GPIO_IN       读取输入
GPIO_OUT      设置输出
GPIO_DIR      选择输入/输出
GPIO_SET      置位某些输出
GPIO_CLEAR    清零某些输出
IRQ_ENABLE    输入中断使能
IRQ_STATUS    输入中断状态
```

### 17.3 为什么 SET/CLEAR 寄存器有用

如果只有 `GPIO_OUT`，软件修改一位可能需要：

```text
读整个寄存器
修改一位
写回整个寄存器
```

中间可能被中断打断。

使用：

```text
GPIO_SET
GPIO_CLEAR
```

可以通过一次写操作原子地修改目标位。

---

## 18. Machine Timer / CLINT：提供操作系统时间

### 18.1 它负责什么

最小 Machine Timer：

```text
mtime       64 位持续递增时间
mtimecmp    64 位截止时间
irq_timer   mtime >= mtimecmp 时拉高
```

软件通过 HXI/MMIO 设置下一次截止时间，Timer 通过专用 IRQ 通知 CPU。

### 18.2 它与现有 Counter 的区别

当前 `counter.sv`：

- 软件 START/STOP；
- 统计累计毫秒；
- 无 `mtimecmp`；
- 无中断输出；
- 不接 CPU `MTIP`。

Machine Timer：

- 持续提供单调时间；
- 软件设置比较值；
- 到点产生机器定时器中断；
- 推动 RT-Thread Tick。

详细内容见 [10：CLINT、Machine Timer 与 RT-Thread Tick](./10_clint_machine_timer_and_rtos_tick.md)。

### 18.3 为什么 Timer IRQ 不经过 PLIC

RISC-V Machine Timer Interrupt 是 hart-local 的标准中断，cause 为 7。

```text
Timer → CPU MTIP
```

UART/GPIO 等外设才通常通过 PLIC/APLIC 汇总为 Machine External Interrupt，cause 为 11。

---

## 19. 中断控制器：管理多个外设事件

### 19.1 为什么需要中断控制器

如果只有 UART 一个外设，可以直接：

```text
UART IRQ → CPU irq_external_i
```

但加入：

```text
UART
GPIO
SPI
DMA
```

CPU 只有一根 External IRQ 输入，不知道是哪一个设备发出的，也无法统一管理优先级和屏蔽。

### 19.2 它负责什么

一个外部中断控制器通常负责：

- 接收多个外设 IRQ；
- 记录 pending；
- 独立 enable/mask；
- 设置优先级；
- 选出当前最高优先级；
- 通知 CPU；
- 让软件 claim 当前中断号；
- 处理完成后 complete。

### 19.3 数据路径与中断路径

事件路径：

```text
UART IRQ ─┐
GPIO IRQ ─┼→ Interrupt Controller → irq_external_i → CPU
SPI IRQ  ─┘
```

配置路径：

```text
CPU → HXI/APB → Interrupt Controller Registers
```

CPU 通过 MMIO 配置 enable/priority，并读取 claim ID；中断控制器通过专用线通知 CPU。

### 19.4 第一版做到多复杂

当前单核 RT-Thread 项目可以分阶段：

#### 最小阶段

```text
Timer IRQ 直连
UART IRQ 直连 external IRQ
软件读取 UART 状态确认来源
```

#### 正式阶段

实现简化 PLIC 风格控制器：

```text
多个外设源
pending
enable
priority
claim/complete
```

如果对外声称“实现 PLIC”，必须核对真正的寄存器和行为规范；否则应称为“项目自定义简化外部中断控制器”。

---

## 20. 时钟系统：让所有状态按确定节奏变化

### 20.1 时钟模块负责什么

- 接收板级晶振；
- 使用 PLL/MMCM 产生需要的频率；
- 提供 CPU 时钟；
- 提供外设或参考时钟；
- 输出 `locked` 表示时钟稳定；
- 必要时做时钟门控。

### 20.2 为什么不能随意在 RTL 中生成新时钟

例如：

```systemverilog
timer_clk <= ~timer_clk;
```

这种普通逻辑生成的时钟会带来：

- 时钟树不受控；
- 时序约束困难；
- 占空比和偏斜问题；
- CDC 数量增加。

常见做法：

```text
同一个系统时钟
+ clock enable
```

例如 Timer 每 50 个系统周期产生一次 `timebase_ce`，但 Timer 寄存器仍由同一个 `soc_clk` 驱动。

### 20.3 什么是时钟域

同一时钟驱动的一组寄存器属于一个 Clock Domain。

当前工程至少涉及：

```text
cpu_clk 域
50 MHz UART/digital-twin 域
```

跨时钟域传输叫 CDC。

---

## 21. 复位系统：把所有模块带到已知初始状态

### 21.1 复位负责什么

复位必须让系统进入可预测状态：

```text
CPU PC = Reset Vector
Cache 全部 Invalid
总线没有未完成事务
UART FIFO 为空
Timer IRQ 不立即触发
中断全部屏蔽
GPIO 输出安全值
```

### 21.2 异步置位、同步释放

FPGA SoC 常用：

```text
复位可以异步拉起
复位释放要在各自时钟域同步
```

原因是解除复位如果靠近时钟边沿，不同触发器可能在不同周期离开复位，造成亚稳态或状态机非法状态。

### 21.3 每个时钟域都要自己的复位同步

```text
global_reset
├─ 同步到 cpu_clk → cpu_rst
└─ 同步到 uart_clk → uart_rst
```

不能只在一个时钟域同步后，直接拿去复位另一个异步时钟域。

### 21.4 复位极性

常见名字：

```text
rst       高有效
rst_n     低有效
resetn    低有效
```

极性必须在端口和文档中一致。Wrapper 可以转换，但不要靠猜。

---

## 22. CDC：跨时钟域为什么危险

### 22.1 亚稳态是什么

一个信号在目标触发器采样边沿附近变化，触发器输出可能短时间既不像 0 也不像 1，称为亚稳态。

### 22.2 单比特慢控制信号

可使用两级同步器：

```text
source_signal
    ↓
sync_ff1
    ↓
sync_ff2
```

适合：

- 电平使能；
- 静态状态；
- 持续时间足够长的 IRQ 电平。

不适合极短脉冲，因为目标时钟可能采不到。

### 22.3 多比特数据

不能简单地每一位各放两个同步器，因为不同位可能在不同周期稳定。

常见方法：

- Gray Code：适合连续计数值；
- 握手：源域保持数据，目标域确认接收；
- Async FIFO：适合连续数据流；
- Snapshot：请求源域锁存一份稳定快照。

当前 `counter.sv` 使用 Gray Code 跨域读取毫秒计数，这正是多位计数器的一种处理方式。

---

## 23. Cache、Store Buffer 与 MMIO 边界

### 23.1 普通 RAM 为什么可以缓存

普通 RAM 的读取没有副作用：

```text
读十次相同地址，通常只是得到相同数据
```

Cache 可以保存副本，提高速度。

### 23.2 外设寄存器为什么不能缓存

读取 UART DATA 可能会：

- 弹出 RX FIFO 一个字节；
- 清除某个状态；

写 Timer compare 会改变中断。

如果 Cache 命中而不访问外设，软件看到的状态就是错的。

### 23.3 Store Buffer 为什么也要识别 MMIO

普通 RAM Store 可以延后、合并。

MMIO Store 可能意味着：

```text
发送一个 UART 字节
清除一个中断
启动 DMA
更新 mtimecmp
```

不能被重复、丢失或越过更早的设备访问。

因此：

```text
地址属性判断
    ↓
普通 RAM → Cache/Store Buffer 优化
MMIO      → bypass，并保证顺序
```

---

## 24. DMA 和 Debug Loader：以后可能增加的 Master

### 24.1 DMA 是什么

DMA 是 Direct Memory Access。

它可以不经过 CPU 执行 Load/Store 指令，直接搬运数据：

```text
UART/SPI/存储器 → DMA → RAM
```

DMA 自己会在 HXI 上发起访问，所以它是 Master。

### 24.2 为什么现在先预留，不急着实现

加入 DMA 会引入：

- 多 Master 仲裁；
- Cache 一致性；
- 内存保护；
- 中断；
- 描述符；
- 错误恢复。

当前目标是 RT-Thread 和 CoreMark，不需要一开始就承担这些复杂度。但 Crossbar 和地址规划可以为第三个 Master 预留接口。

### 24.3 Debug Loader 是什么

它可能通过 UART/JTAG 把程序写入 BRAM，再让 CPU 从指定地址启动。

如果它直接访问 RAM，也是一个 Master。

这会影响：

- Code BRAM 是否运行时可写；
- I-cache 是否需要 `FENCE.I`/invalidate；
- CPU 运行时与 Loader 同时访问怎样仲裁。

---

## 25. 三条最重要的数据路径

### 25.1 取指路径

```text
PC
 ↓
IFU / I-cache
 ↓ HXI-I request
Crossbar
 ↓
Code BRAM Adapter
 ↓
Code BRAM
 ↓ instruction response
CPU
```

需要保证：

- 地址对齐；
- BRAM 读延迟正确；
- access fault 可返回；
- 分支 flush 后旧响应不会被误用；
- 如果有 I-cache，`FENCE.I` 语义明确。

### 25.2 普通 Load/Store 路径

```text
LSU
 ↓
D-cache / Store Buffer
 ↓ miss/refill/write-through
HXI-D
 ↓
Crossbar
 ↓
Data BRAM Adapter
 ↓
Data BRAM
```

需要保证：

- byte/halfword/word；
- 地址对齐；
- sign extension；
- `wstrb`；
- Cache hit/miss；
- 精确异常；
- Store 不重复。

### 25.3 MMIO 路径

```text
LSU 计算出 UART/Timer 地址
 ↓
识别为 non-cacheable
 ↓ bypass D-cache
HXI-D
 ↓
Crossbar
 ├─ Timer Slave
 └─ HXI-to-APB → UART/GPIO
 ↓
设备寄存器响应
```

需要保证：

- 不缓存；
- 不预取；
- 不丢写；
- 不重复执行副作用；
- 错误能够返回；
- 软件 `volatile` 访问真正到达设备。

---

## 26. 中断路径的完整过程

以 UART 为例：

```text
电脑发送字符
    ↓
UART RX 引脚收到串行位
    ↓
UART 恢复出一个字节，放进 RX FIFO
    ↓
UART 置位 rx_irq
    ↓
Interrupt Controller 记录 pending
    ↓
CPU irq_external_i = 1
    ↓
CPU 检查 mstatus.MIE、mie.MEIE
    ↓
精确进入 mtvec
    ↓
Trap 入口保存寄存器
    ↓
中断处理程序读取 claim ID
    ↓
UART 驱动读取字符
    ↓
送给 RT-Thread 串口框架/FinSH
    ↓
complete 中断并 mret
```

这条链包含三个层次：

```text
外设硬件事件
CPU 架构级 Trap
操作系统驱动和上层服务
```

任何一层缺失，FinSH 都不能正常收到字符。

---

## 27. SoC 中哪些是软件可见的，哪些不是

### 27.1 软件直接可见

软件能通过地址或 CSR 访问：

- RAM/ROM 地址；
- UART/GPIO/Timer 寄存器；
- 中断控制器寄存器；
- CPU CSR；
- `cycle/instret`；
- 异常和中断 cause。

### 27.2 软件间接感知

软件看不到模块名，但会感受到行为：

- Crossbar 仲裁延迟；
- BRAM 读取延迟；
- Cache miss；
- Bridge 等待；
- Store Buffer 顺序；
- Clock 频率。

### 27.3 软件通常完全不应依赖

- Xilinx BRAM primitive 名字；
- RTL 文件夹层次；
- 内部状态机编码；
- 仲裁器内部寄存器名；
- PLL IP 实例名。

硬件结构可以变化，只要软件可见契约不变，BSP 和应用不应修改。

---

## 28. 当前工程与目标 SoC 的对应关系

| 组件 | 当前工程 | 目标状态 |
| --- | --- | --- |
| FPGA 顶层 | 有 `top.sv`、PLL、digital-twin UART | 保留板级层，明确 Console UART 方案 |
| SoC 集成 | 有 `student_top.sv` | 演进为清晰的 `soc_top/cpu_subsystem` |
| CPU 指令接口 | 直接 `irom_addr/data/ena` | 可先保留，后续规范为 HXI-I |
| CPU 数据接口 | 已有 request/response 雏形 | 增加 error，正式冻结协议 |
| D-cache | CPU 内部已有 | 保留，完善 MMIO bypass 与异常 |
| I-cache | 当前没有完整 I-cache | 根据取指瓶颈决定 |
| Interconnect | `SocMemBridge` 是单 Master 地址分流 | 演进为 2×N HXI Crossbar |
| Code Memory | `IROM_0` BRAM | 扩容，明确 `.rodata` 与 D-side 访问 |
| Data Memory | `DramBramAdapter`/BRAM | 冻结容量、读延迟和写掩码 |
| UART | 主要服务 digital twin | 增加 CPU MMIO Console UART |
| GPIO | 虚拟 SW/KEY/LED/SEG | 作为 APB GPIO 规范化 |
| Counter | 有 START/STOP 毫秒计数 | 可保留为测量外设 |
| Machine Timer | 没有 | 新增 `mtime/mtimecmp/MTIP` |
| CPU 中断 | 顶层无 IRQ 输入 | 增加软件/Timer/外部 IRQ |
| 外部中断控制器 | 没有 | UART 单路直连起步，后续简化 PLIC |
| HXI-to-APB | 没有正式桥 | 新增桥和 APB 子系统 |
| Default Slave | 不完整 | 非法地址必须返回 error |
| Clock/Reset | 已有两个域和同步逻辑 | 统一文档、每域同步释放、CDC 审查 |

这里最重要的变化不是“增加很多文件”，而是把当前的：

```text
CPU + 专用存储接口 + 若干地址判断
```

演进成：

```text
CPU 子系统 + 正式互联 + 统一存储从设备 + 外设子系统 + 中断系统
```

---

## 29. 为什么要这样划分模块

### 29.1 为了独立验证

可以分别测试：

```text
CPU 是否正确发请求
Crossbar 是否正确路由
BRAM Adapter 是否处理一拍读延迟
Timer 是否正确产生 IRQ
UART 是否正确收发
```

不需要每次都启动完整 RT-Thread 才知道哪里错了。

### 29.2 为了替换实现

接口不变时可以替换：

```text
行为 RAM → Xilinx BRAM IP
简单双口 RAM → 更大存储器
自定义 UART → 另一版 UART
单 Master Bridge → Crossbar
```

### 29.3 为了多人协作

三个人可以按稳定边界并行：

```text
队长：
    CPU/SoC 接口、Memory Map、中断架构、最终集成

队员 A：
    HXI Crossbar、BRAM Adapter、Timer/外设 RTL

队员 B：
    RT-Thread BSP、UART/Timer 驱动、构建与测试
```

前提是先冻结：

```text
端口
地址
时序
错误
中断
复位
```

否则每个人都会按自己的理解修改同一边界。

### 29.4 为了让软件不依赖 RTL 细节

软件只应依赖：

```text
地址
寄存器
中断号
时钟频率
访问语义
```

这些是硬件—软件契约。

---

## 30. 每个组件的接口文档应该写什么

以后你们定义任何模块，都可以按下面模板。

### 30.1 基本信息

```text
模块名
功能
所在层次
时钟域
复位极性
```

### 30.2 输入输出

每个端口说明：

```text
名称
方向
位宽
时钟域
含义
何时有效
复位行为
```

### 30.3 协议

```text
请求怎样握手
响应怎样握手
是否允许背压
最多几笔 outstanding
响应顺序
错误表示
```

### 30.4 地址和寄存器

```text
基地址
窗口大小
每个寄存器偏移
R/W/RO/W1C/WO
复位值
副作用
```

常见缩写：

| 缩写 | 含义 |
| --- | --- |
| RO | 只读 |
| RW | 可读写 |
| WO | 只写 |
| W1C | 写 1 清零 |
| W1S | 写 1 置位 |
| RC | 读取后清零 |

### 30.5 中断

```text
中断来源
电平还是脉冲
pending 在哪里保存
enable/mask 在哪里
怎样清除
CPU cause/IRQ 号
多个来源如何仲裁
```

### 30.6 异常情况

```text
非法地址
非对齐访问
不支持的写掩码
FIFO 满/空
同时读写
复位期间请求
```

---

## 31. 对当前阶段最合适的实现顺序

### 阶段 1：冻结 SoC 契约

先写清：

- HXI 请求/响应；
- Memory Map；
- Code/Data BRAM 容量；
- MMIO 属性；
- Timer 寄存器；
- CPU IRQ；
- UART 寄存器；
- 复位和时钟。

### 阶段 2：完成最小 HXI

```text
测试 Master
    → Crossbar
       ├─ RAM Model
       └─ Default Slave
```

### 阶段 3：接 BRAM Adapter

验证：

- 同步读延迟；
- byte write；
- 边界地址；
- 请求背压；
- 错误响应。

### 阶段 4：接 CPU，不开 Cache

先跑：

- 取指；
- Load/Store；
- UART 轮询；
- 裸机程序。

### 阶段 5：接 Timer 和 CPU 中断

先完成裸机周期中断，再接 RT-Thread Tick。

### 阶段 6：接 D-cache

重点验证：

- 普通 RAM；
- miss/refill；
- Store；
- MMIO bypass；
- 中断期间未完成访问。

### 阶段 7：接 APB UART/GPIO

再启动：

- RT-Thread；
- FinSH；
- CoreMark 命令。

### 阶段 8：再做性能和扩展

根据测量结果决定：

- I-cache；
- 更大 D-cache；
- burst；
- 多 outstanding；
- DMA；
- 完整外部中断控制器。

---

## 32. 你现在应该形成的 SoC 思维

看到一个 SoC 模块时，不要先问：

```text
“这个模块有多少行 RTL？”
```

先问六个问题：

1. 它解决什么系统问题？
2. 谁会主动访问它？
3. 它通过什么接口收发数据？
4. 它会不会主动产生中断？
5. 软件通过什么地址或 CSR 看见它？
6. 访问失败或等待时，系统怎样结束？

例如 Machine Timer：

```text
问题：
    操作系统需要时间和周期事件

访问者：
    CPU D-side Master

数据接口：
    HXI MMIO，读 mtime、写 mtimecmp

事件接口：
    irq_timer

软件可见：
    Timer 地址、TIMEBASE_FREQ、mcause 7

失败/等待：
    非法访问返回总线错误，IRQ 保持到比较值更新
```

例如 UART：

```text
问题：
    SoC 需要和电脑交换字符

访问者：
    CPU 通过 HXI→APB 访问

数据接口：
    DATA/STATUS/CTRL 寄存器和 RX/TX 引脚

事件接口：
    RX/TX IRQ

软件可见：
    UART 基地址、寄存器、波特率、中断号

失败/等待：
    FIFO 满/空、错误状态、总线响应
```

当你能用这个框架描述每个模块时，就已经从“会写一个 CPU 模块”迈到了“会规划一套 SoC”。

---

## 33. 总结表

| 组件 | 一句话职责 | 主要接口 | 为什么独立 |
| --- | --- | --- | --- |
| FPGA Top | 连接真实板级时钟和引脚 | 时钟、复位、UART/GPIO pins | 隔离具体开发板 |
| SoC Top | 实例化并连接整个系统 | 内部模块连接、外部 I/O | 表达系统集成关系 |
| CPU Core | 执行指令并处理 Trap | I/D memory、IRQ、debug | 与具体外设解耦 |
| CPU Wrapper | 做边界适配 | reset、bus、IRQ 转换 | 保持 Core 接口稳定 |
| Cache | 保存常用内存副本 | CPU-side、memory-side | 降低访存延迟 |
| HXI Crossbar | 路由多主多从事务 | request/response | 统一系统通信 |
| Address Decoder | 根据地址选择设备 | addr → select | 统一地址空间 |
| Arbiter | 解决资源争用 | 多请求 → 一个 grant | 避免冲突和饥饿 |
| Default Slave | 结束非法地址访问 | error response | 防止死锁和静默错误 |
| Code BRAM | 保存指令和只读映像 | BRAM/HXI read | FPGA 内部程序存储 |
| Data BRAM | 保存变量、堆和栈 | BRAM/HXI read/write | 提供运行时存储 |
| BRAM Adapter | 转换总线与 BRAM 时序 | HXI ↔ BRAM native | 隔离存储实现 |
| HXI-to-APB | 转换两种协议 | HXI Slave/APB Master | 复用简单外设总线 |
| UART | 串口字节收发 | APB、RX/TX、IRQ | Console/FinSH |
| GPIO | 控制和读取引脚 | APB、pins、IRQ | 通用板级 I/O |
| Machine Timer | 单调计时并产生 Tick IRQ | MMIO、irq_timer | RT-Thread 时间基础 |
| Interrupt Controller | 管理多个外设 IRQ | MMIO、source IRQ、CPU IRQ | 屏蔽、优先级、识别来源 |
| Clock/Reset | 提供可靠时序和初始状态 | clocks、resets、locked | 所有时序逻辑的基础 |
| CDC Logic | 安全跨越时钟域 | synchronizer/FIFO/handshake | 防止亚稳态和数据撕裂 |
| DMA（后续） | 无需 CPU 搬运数据 | HXI Master、IRQ | 提高 I/O 效率 |

---

## 34. 参考资料

- [09：HXI 片上互联、BRAM 存储与 Cache 架构说明](./09_hxi_interconnect_bram_cache_architecture.md)
- [10：CLINT、Machine Timer 与 RT-Thread Tick](./10_clint_machine_timer_and_rtos_tick.md)
- [Arm：Introduction to AMBA AXI4](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf)——包含 AMBA、AHB、AHB-Lite、AXI 和 APB 的定位。
- [RISC-V Privileged Architecture](https://docs.riscv.org/reference/isa/priv/machine.html)——CSR、Trap、机器模式中断与 Machine Timer。
- [RISC-V Advanced Interrupt Architecture](https://docs.riscv.org/reference/aia/intro.html)——RISC-V 外部中断控制和 local interrupt 的系统定位。
- [AMD Block Memory Generator PG058](https://docs.amd.com/v/u/en-US/pg058-blk-mem-gen)——FPGA BRAM 类型、接口和 IP 配置。

