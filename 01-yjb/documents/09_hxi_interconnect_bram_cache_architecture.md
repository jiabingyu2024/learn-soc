# HXI 片上互联、BRAM 存储与 Cache 的 SoC 架构说明

> 本文面向当前 `superScalar` 自研 RV32 SoC。  
> 参考材料：去年的优秀作品《CICC0905422 全国总决赛技术文档》、当前 `dev-v4.0` RTL，以及 Arm、AMD、RISC-V 官方资料。  
> 本文目标不是给出一个只能点灯的最小 Demo，而是规划一套适合继续接入 RT-Thread、CoreMark、UART、Timer、GPIO、PLIC/中断控制器，并可继续增加 DMA/调试主设备的片上系统架构。

本文主要回答：

1. 优秀作品中的 HXI 是否就是 AHB-to-APB；
2. AHB、APB 在 SoC 中究竟怎样连接；
3. 如果继续采用 HXI，多主多从互联应怎样设计；
4. iRAM、dRAM 使用 FPGA BRAM 时，软件镜像怎样进入 BRAM；
5. BRAM 在 SoC 内怎样连接 CPU、总线与 Cache；
6. Cache 应放在 CPU 核内部、CPU 外部还是 BRAM 前面；
7. 当前工程应该从现状演进到什么结构。

---

## 1. 先给出最重要的判断

### 1.1 HXI 不是 AHB，也不是 AHB-to-APB

去年的优秀作品把 HXI 定义为：

```text
HXI = Hybrid eXtensible Internal Bus
```

其特点是：

- `valid/ready` 握手；
- 统一的读写请求接口；
- 32 位地址和数据；
- 4 位字节写掩码；
- 多主、多从；
- `NM × NS` 交叉互联；
- 请求由 `1-to-N` 分发；
- 响应由 `N-to-1` 返回；
- 多个主设备访问同一从设备时进行仲裁；
- 不同主设备访问不同从设备时可以并行。

优秀作品中配置为：

```text
主设备 NM = 2
├─ M0：IFU，负责取指
└─ M1：LSU，负责数据和 MMIO 访问

从设备 NS = 6
├─ S0：UART
├─ S1：GPIO
├─ S2：PLIC
├─ S3：CLINT
├─ S4：ROM
└─ S5：RAM
```

该文档自己将它描述为“AXI-Lite 风格的一次一拍地址握手与数据握手”，而不是 AMBA AHB。

准确地说：

```text
HXI 是自定义请求/响应总线
AHB 是 Arm AMBA 标准总线
APB 是 Arm AMBA 外设总线
AHB-to-APB 是两种标准协议之间的桥
```

因此不能因为 HXI 也有地址、数据和握手，就把它称为 AHB。

### 1.2 HXI 与 AHB-to-APB 相似的是“系统分层思想”

两者都可以形成：

```text
CPU/Cache
    ↓
主干互联
    ↓
存储器、桥、外设
```

经典 AMBA 系统是：

```text
CPU AHB Master
    ↓
AHB Interconnect
    ├─ AHB SRAM
    ├─ AHB ROM
    └─ AHB-to-APB Bridge
            └─ APB UART/GPIO/Timer
```

采用 HXI 后可以写成：

```text
CPU HXI Masters
    ↓
HXI Crossbar
    ├─ HXI-to-BRAM Adapter → Code BRAM
    ├─ HXI-to-BRAM Adapter → Data BRAM
    └─ HXI-to-APB Bridge
            └─ APB UART/GPIO/Timer
```

所以可以说：

> HXI-to-APB 在整个 SoC 中扮演的角色，类似常见系统里的 AHB-to-APB；但它的上游协议是 HXI，不是 AHB。

### 1.3 对当前项目的推荐结论

我建议保留 HXI 思路，但把它从“若干 valid/ready 信号”提升为有明确规范的 SoC 主干互联：

```text
CPU 子系统
├─ Pipeline / Frontend / LSU
├─ I-cache（可选但预留）
├─ D-cache
├─ HXI-I Master
└─ HXI-D Master

HXI Crossbar
├─ Code BRAM Slave
├─ Data BRAM Slave
├─ HXI-to-APB Bridge
├─ CLINT/Machine Timer Slave
├─ Interrupt Controller Slave
└─ Default Error Slave

APB Peripheral Subsystem
├─ UART
├─ GPIO
├─ Performance Counter Control
├─ SPI（后续）
└─ 其他低速寄存器外设
```

第一版互联可以限制为：

- 每个 Master 同时最多一个未完成事务；
- 每个 Slave 同时服务一个 Master；
- 不支持乱序响应；
- Cache Line refill 用多个单拍请求完成；
- 支持不同 Master 并行访问不同 Slave；
- 同一 Slave 上采用轮询或确定的固定优先级仲裁；
- 所有请求都有明确完成或错误响应。

这不是“玩具总线”。只要协议、仲裁、响应归属、错误处理和验证写清楚，它足以支撑单核 RV32、RTOS、Cache、MMIO、调试端口和简单 DMA。

---

## 2. “TL 层次”可能指什么

这句话可能有两种含义。

### 2.1 如果你指的是 RTL 层次

HXI 就是 SoC RTL 内部的接口规范。它位于：

```text
CPU/Cache RTL
    ↕ HXI
Interconnect RTL
    ↕ HXI Slave Port
Memory/Bridge RTL
```

它不会出现在芯片外部引脚，也不会被 C 程序直接看到。软件只看到统一地址空间。

### 2.2 如果你指的是 TileLink

TileLink 也是一种片上互联协议，常见于 Rocket Chip、Chipyard 等 RISC-V 项目。最简单的 TileLink-UL 通常有：

- A 请求通道；
- D 响应通道；
- opcode；
- size；
- source；
- address；
- mask；
- data；
- `valid/ready`。

HXI 在“请求通道 + 响应通道 + valid/ready + mask”这一思想上与 TileLink-UL 有相似之处，但没有按照 TileLink 的字段和规则实现，因此也不能称为 TileLink。

当前项目没有必须兼容 Rocket/Chipyard IP 的要求，因此没有必要为了名字更标准而强行改成 TileLink。更重要的是把自己的 HXI 规范写完整并验证。

---

## 3. AHB 在 SoC 中究竟怎样工作

### 3.1 AHB 的定位

AHB 是 AMBA 中面向片上高性能访问的总线，常用于连接：

- CPU；
- SRAM/ROM；
- DMA；
- 调试模块；
- 高带宽外设；
- APB Bridge。

AHB 与 HXI 最大的接口差异是：

```text
HXI：常见实现是 request/response valid-ready
AHB：地址控制相位与数据相位流水重叠，使用 HREADY 延长传输
```

### 3.2 一个 AHB-Lite Master 的典型信号

常见信号包括：

```text
HADDR       地址
HTRANS      当前是否为有效传输，以及 NONSEQ/SEQ 等类型
HWRITE      读或写
HSIZE       byte/halfword/word 等访问宽度
HBURST      突发类型
HPROT       访问属性
HWDATA      写数据

HRDATA      读数据
HREADY      当前传输是否完成
HRESP       正常或错误响应
```

Slave 选择通常由互联译码产生：

```text
HSEL_SRAM
HSEL_ROM
HSEL_APB_BRIDGE
...
```

### 3.3 AHB 的地址相位和数据相位

一次简单读访问可以理解为：

```text
周期 N
Master 输出 HADDR、HTRANS、HWRITE、HSIZE

周期 N+1
Slave 返回 HRDATA
HREADY=1 表示本次传输完成
```

由于地址相位和上一笔数据相位可以重叠，AHB 具有流水化特点：

```text
周期 1：请求 A 的地址
周期 2：返回 A 的数据，同时请求 B 的地址
周期 3：返回 B 的数据，同时请求 C 的地址
```

如果 Slave 没准备好，就拉低 `HREADY`，整个传输保持等待。

这与 HXI 的：

```text
req_valid && req_ready
rsp_valid && rsp_ready
```

不是同一种时序语义。

### 3.4 AHB 与 AHB-Lite 的区别

AHB-Lite 面向单 Master 接口，去掉了完整 AHB 中与多 Master 仲裁有关的部分。

但这不代表整个 SoC 只能有一个 Master。常见做法是：

```text
CPU-I AHB-Lite Master ─┐
CPU-D AHB-Lite Master ─┼─ AHB Bus Matrix ─ 多个 Slave
DMA AHB-Lite Master ───┘
```

每个输入端口仍是 AHB-Lite 风格，由 Bus Matrix 在内部完成多主多从连接和仲裁。

这与 HXI Crossbar 的系统目标相同，但协议细节不同。

### 3.5 AHB SRAM 怎样接

AHB SRAM Controller/Adapter 负责把 AHB 相位转换为 SRAM 或 BRAM 原生端口：

```text
AHB Slave Port
├─ 接收 HADDR/HWRITE/HSIZE/HWDATA
├─ 产生 BRAM ena/wea/addra/dina
├─ 等待同步读数据
└─ 产生 HRDATA/HREADY/HRESP
```

BRAM 读需要一个或多个时钟周期时，Adapter 用 `HREADY=0` 插入等待。

### 3.6 为什么还要 APB

UART、GPIO、Timer 控制寄存器通常不需要：

- 流水访问；
- Cache Line burst；
- 很高吞吐；
- 多笔未完成事务。

如果每个小外设都完整实现 AHB，接口逻辑会重复而且复杂。因此使用更简单的 APB。

---

## 4. APB 与 AHB-to-APB Bridge

### 4.1 APB 的典型信号

```text
PADDR
PSEL
PENABLE
PWRITE
PWDATA
PSTRB       APB4 字节写掩码
PRDATA
PREADY
PSLVERR
```

### 4.2 一次 APB 访问

APB 不做地址/数据流水重叠。一次传输至少分两阶段。

#### SETUP 阶段

```text
PSEL    = 1
PENABLE = 0
PADDR/PWRITE/PWDATA/PSTRB 有效
```

#### ACCESS 阶段

下一周期：

```text
PSEL    = 1
PENABLE = 1
```

若：

```text
PREADY = 1
```

传输在该周期完成；否则保持 ACCESS 状态继续等待。

### 4.3 AHB-to-APB Bridge 的内部流程

Bridge 上游表现为一个 AHB Slave，下游表现为 APB Master：

```text
AHB 请求
    ↓
Bridge 锁存地址、方向、写数据和字节使能
    ↓
APB SETUP
    ↓
APB ACCESS
    ↓ 等待 PREADY
读取 PRDATA/PSLVERR
    ↓
向 AHB 返回 HRDATA/HRESP，并释放 HREADY
```

### 4.4 HXI-to-APB Bridge 完全可以采用相同分层

只是把上游换成：

```text
hxi_req_valid
hxi_req_ready
hxi_req_addr
hxi_req_write
hxi_req_wdata
hxi_req_wstrb

hxi_rsp_valid
hxi_rsp_ready
hxi_rsp_rdata
hxi_rsp_err
```

Bridge 状态机可以规划为：

```text
IDLE
  接收一笔 HXI 请求并锁存
    ↓
APB_SETUP
  PSEL=1, PENABLE=0
    ↓
APB_ACCESS
  PSEL=1, PENABLE=1，等待 PREADY
    ↓
HXI_RESPONSE
  保持 rsp_valid，直到 rsp_ready
    ↓
IDLE
```

因此，我对当前项目的建议不是“必须换成 AHB”，而是：

> 采用 HXI Crossbar 作为主干，再实现 HXI-to-APB Bridge，得到与成熟 MCU SoC 类似的两级总线结构。

---

## 5. HXI 应该定义成什么样

优秀作品已经给出了方向，但要用于后续长期集成，还需要把隐含规则变成正式协议。

### 5.1 推荐的单拍请求通道

```systemverilog
req_valid
req_ready
req_addr[31:0]
req_write
req_wdata[31:0]
req_wstrb[3:0]
req_size[1:0]
req_instr
```

字段含义：

| 信号 | 含义 |
| --- | --- |
| `req_valid` | Master 当前提供有效请求 |
| `req_ready` | Interconnect/Slave 可以接收请求 |
| `req_addr` | 字节地址 |
| `req_write` | 1 为写，0 为读 |
| `req_wdata` | 写数据 |
| `req_wstrb` | 每一位对应一个写入字节 |
| `req_size` | byte、halfword、word；也用于错误检查 |
| `req_instr` | 可选，标记取指访问 |

请求只在：

```text
req_valid && req_ready
```

同一上升沿成立时被接收。

### 5.2 推荐的响应通道

```systemverilog
rsp_valid
rsp_ready
rsp_rdata[31:0]
rsp_err
```

读请求返回 `rsp_rdata`；写请求也建议给出完成响应，以便：

- 确认真正写入；
- 传播非法地址或 Slave 错误；
- 为 StoreBuffer 精确提交提供明确完成边界；
- 防止“请求被接收”等同于“写操作已完成”的歧义。

### 5.3 必须写进协议的保持规则

当：

```text
req_valid = 1
req_ready = 0
```

Master 必须保持：

```text
req_valid
req_addr
req_write
req_wdata
req_wstrb
req_size
```

不变，直到握手。

响应端同理：

```text
rsp_valid = 1
rsp_ready = 0
```

Slave/Interconnect 必须保持响应数据和错误位不变。

这些规则必须写断言验证，不能只依赖模块“通常会这样做”。

### 5.4 第一版限制一个 Outstanding 是合理的

第一版建议：

```text
每个 Master 最多一笔尚未收到响应的请求
```

好处是：

- 不需要 Transaction ID；
- 响应顺序天然确定；
- Crossbar 只需记录每个 Slave 当前属于哪个 Master；
- Cache refill 状态机简单；
- 更容易验证精确异常和 MMIO 顺序。

后续需要提高带宽时，再增加：

- `req_id/rsp_id`；
- 多笔 outstanding；
- burst；
- reorder buffer。

一开始就同时做这些，会把主要工作从 CPU/RTOS 转移到复杂总线验证上。

### 5.5 Cache Line refill 怎样走 HXI

假设 Cache Line 为 16 Byte，数据宽度为 32 位：

```text
一条 Cache Line = 4 个 word
```

第一版可依次发起：

```text
base + 0
base + 4
base + 8
base + 12
```

共四笔单拍读事务。

不必为了 Cache Line 立刻增加 burst。单拍 refill 带宽较低，但接口和验证清晰。等系统稳定后再增加：

```text
req_len
rsp_last
```

或专用 burst 扩展。

---

## 6. 多主多从 Crossbar 怎样工作

### 6.1 请求路径

每个 Master 先进行地址译码：

```text
M0 请求
├─ 命中 Code BRAM → S0
├─ 命中 Data BRAM → S1
├─ 命中 APB Window → S2
└─ 未命中 → Default Slave
```

概念上得到一个选择向量：

```text
m_decode[M][S]
```

每个 Slave 再观察有哪些 Master 请求自己：

```text
S0 requesters = {M0, M1, M2...}
```

### 6.2 同时访问不同 Slave

例如：

```text
I-cache Miss → Code BRAM
D-cache Store → Data BRAM
```

两者目标不同，可以同周期被接收：

```text
M0 → S0
M1 → S1
```

这就是 Crossbar 相比单共享总线的重要价值。

### 6.3 同时访问同一 Slave

例如：

```text
IFU 想从 Data BRAM 执行代码
LSU 同时访问 Data BRAM 数据
```

需要仲裁：

```text
M0 ─┐
    ├─ Arbiter → Data BRAM
M1 ─┘
```

推荐：

- 第一版可以使用 Round-Robin；
- 或固定优先级，但必须分析饥饿；
- 仲裁结果必须保持到当前事务完成；
- 有响应之前不能把 Slave 所有权切给另一 Master。

如果采用固定优先级：

```text
LSU > IFU
```

大量数据访问可能让取指长期得不到服务；反过来又可能让数据访问饥饿。因此 Round-Robin 更稳妥。

### 6.4 响应路径

Crossbar 在请求握手时记录：

```text
slave_owner[S] = M
```

Slave 返回响应后：

```text
S.rsp → owner Master
```

不能根据“当前地址”组合选择返回路径，因为响应可能比请求晚多个周期，此时 Master 地址早已变化。

这是简单 SoC 中最常见的错误之一：

```text
只对请求做地址译码
却没有保存响应应该返回哪个 Master
```

### 6.5 Default Slave

未映射地址不能永久没有响应，否则 CPU 会死锁。

Default Slave 应在确定延迟后返回：

```text
rsp_valid = 1
rsp_err   = 1
rsp_rdata = 某个固定调试值
```

CPU 收到错误后应进入 Load/Store access fault 或 Instruction access fault。

---

## 7. 推荐的主从连接矩阵

初始可以定义三个 Master：

```text
M0：I-cache/IFU
M1：D-cache/LSU
M2：Debug Loader 或 DMA（先预留）
```

推荐 Slave：

```text
S0：Code BRAM
S1：Data BRAM
S2：HXI-to-APB Bridge
S3：Machine Timer/CLINT
S4：Interrupt Controller
S5：Default Error Slave
```

连接矩阵建议：

| Master \ Slave | Code BRAM | Data BRAM | APB Bridge | CLINT | IRQ Ctrl | Default |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| I-side | 是 | 可选 | 否 | 否 | 否 | 是 |
| D-side | 至少可读 | 是 | 是 | 是 | 是 | 是 |
| Debug/DMA | 可选读写 | 可选读写 | 可选 | 可选 | 可选 | 是 |

### 7.1 为什么 D-side 至少要能读取 Code BRAM

C 程序中的：

- 字符串常量；
- `const` 数组；
- 跳转表；
- 只读查找表；
- `.rodata`；

虽然和代码一起存放在镜像里，但它们由普通 Load 指令读取，走的是 LSU/D-side，不是 IFU。

如果链接脚本把 `.rodata` 放在 Code BRAM，而 D-side 完全不能访问 Code BRAM，程序会在读取常量时失败。

有三种处理方式：

1. D-side 允许只读访问 Code BRAM；
2. 把 `.rodata` 全部放到 Data BRAM；
3. 使用统一存储器。

推荐第一种，因为最符合常见固件布局。

### 7.2 为什么 I-side 可以选择访问 Data BRAM

若未来需要：

- Bootloader 把程序搬入 RAM 后执行；
- 调试器下载代码到 RAM；
- 动态加载模块；
- RAM 中断向量或 RAM 函数；

IFU 就需要从 Data BRAM 取指。

第一版若不需要，可以禁止并返回 Instruction access fault；但地址属性要明确，不要无意中允许。

---

## 8. iRAM、IROM、Code BRAM 是不是同一个东西

### 8.1 BRAM 是 FPGA 物理资源

Kintex-7 芯片内部有固定的 Block RAM 资源。RTL 或 Vivado IP 可以把它配置成：

- 单口 RAM；
- Simple Dual-Port RAM；
- True Dual-Port RAM；
- ROM；
- FIFO；
- 不同宽度和深度；
- 带字节写使能的存储器。

BRAM 本身不是天然的 ROM。是否能写由 RTL 是否暴露写端口、写使能是否接通决定。

### 8.2 IROM 是逻辑角色

如果一块 BRAM：

- 上电时带有程序初始化内容；
- CPU 只能读；
- 写使能永远关闭；

它在 SoC 中就扮演 IROM/Code ROM。

### 8.3 iRAM 是可写的代码存储器

如果同一块 BRAM：

- 可以由 Debug Master、Bootloader 或 D-side 写入；
- IFU 可以从中取指；

它更适合叫：

```text
Instruction RAM
Code RAM
```

### 8.4 本项目建议使用 `code_bram` 命名

物理实现写：

```text
code_bram：FPGA BRAM
```

软件属性再写：

```text
运行时 CPU 默认只读、可执行
调试/下载端口可选择写入
```

这样比 `IROM_0` 更准确，也为后续下载镜像保留空间。

---

## 9. 程序是怎样进入 BRAM 的

### 9.1 软件构建阶段

RT-Thread、BSP、CoreMark 和应用程序一起经过：

```text
.c/.S
  ↓ 交叉编译
.o
  ↓ link.lds
firmware.elf
  ↓ objcopy
firmware.bin
  ↓ 镜像转换工具
firmware.hex / firmware.coe / firmware.mem
```

链接脚本决定每个 section 的运行地址：

```text
.text/.rodata
.data
.bss
stack
heap
```

### 9.2 FPGA 配置阶段

BRAM 可以由 bitstream 预初始化。

常见流程：

```text
firmware.coe
    ↓ 配置 Block Memory Generator
BRAM INIT 属性
    ↓ Vivado 生成 bitstream
FPGA 配置
    ↓
BRAM 自动拥有初始程序内容
```

仿真时则通常：

```systemverilog
$readmemh("firmware.hex", mem);
```

需要保证两种模型的：

- 字节序；
- word 排列；
- 起始地址；
- 深度；
- 空洞填充值；

完全一致。

### 9.3 上电执行

FPGA 配置完成后：

```text
复位 PC = Code BRAM 基地址
    ↓
IFU 发起取指
    ↓
I-cache 命中或向 HXI 请求
    ↓
HXI 译码到 Code BRAM Slave
    ↓
BRAM 同步读出第一条指令
    ↓
CPU 进入 start.S
```

### 9.4 `.data` 和 `.bss` 怎么处理

采用分离 Code BRAM/Data BRAM 时，常见做法是：

```text
.text/.rodata
    直接留在 Code BRAM

.data 初值
    镜像副本保存在 Code BRAM
    start.S 复制到 Data BRAM

.bss
    start.S 清零 Data BRAM 对应区域

stack/heap
    只在 Data BRAM 中保留空间
```

所以链接脚本需要区分：

```text
.data 的加载地址 LMA
.data 的运行地址 VMA
```

启动代码通常使用：

```text
_sidata
_sdata
_edata
_sbss
_ebss
```

完成复制和清零。

### 9.5 是否每次改软件都要重新综合

最直接的 COE/IP 初始化流程经常需要更新 bitstream。

后续可以增加：

- Vivado `updatemem`/MEM 文件更新流程；
- JTAG-to-BRAM 下载；
- UART Bootloader；
- Debug Master；
- SPI Flash Boot。

这些方式只改变 BRAM 内容或启动加载过程，不必每次修改 CPU RTL。

---

## 10. BRAM 怎样接入 HXI

### 10.1 不要让 Crossbar 直接依赖 Xilinx IP 端口

应增加统一 Adapter：

```text
HXI Slave Port
    ↓
hxi_bram_slave
    ↓
BRAM Native Port
```

这样：

- Crossbar 不知道 BRAM 读延迟；
- 仿真行为模型和 Vivado IP 共用相同 HXI 接口；
- 更换单口/双口 BRAM 时只改 Adapter；
- 可以统一处理字节写、错误和响应。

### 10.2 BRAM 原生接口

以 32 位数据为例：

```text
clka
ena
addra[word_addr]
dina[31:0]
wea[3:0]
douta[31:0]
```

地址转换为：

```text
word_addr = (req_addr - BASE_ADDR) >> 2
```

`req_wstrb[3:0]` 可映射到 BRAM 的四个 byte write enable。

### 10.3 同步读的状态机

假设 BRAM 原生读延迟为 1 周期：

```text
IDLE
  req_valid && req_ready
  锁存请求，驱动 ena/address
    ↓
READ_WAIT
  等待 BRAM dout 有效
    ↓
RESP
  rsp_valid=1
  保持数据直到 rsp_ready
    ↓
IDLE
```

如果启用 BRAM 输出寄存器，可能变成 2 周期。Adapter 应通过参数固定并明确延迟，不要让 CPU 猜测。

### 10.4 写访问

写请求握手后：

```text
ena  = 1
wea  = req_wstrb
addr = word_addr
dina = req_wdata
```

在时钟沿完成写入。

之后向 HXI 返回写响应。不要让 Master 在 Slave 尚未真正接受请求时提前认为 Store 完成。

### 10.5 非对齐访问

必须在 CPU/LSU 与总线规范之间明确责任：

方案 A：

```text
LSU 保证所有送到 HXI 的访问都已对齐
```

方案 B：

```text
HXI/Adapter 支持跨 word 的非对齐访问
```

第一版推荐方案 A：

- byte 可位于任意地址；
- halfword 要求地址低位符合约束；
- word 要求四字节对齐；
- 不支持的非对齐访问由 CPU 产生异常；
- `wdata/wstrb` 在 LSU 中完成移位和掩码。

否则一个跨 32 位边界的 Store 需要拆成两笔总线事务，精确异常和 StoreBuffer 都会复杂很多。

---

## 11. Code BRAM 与 Data BRAM 用单口还是双口

### 11.1 两块独立 BRAM：推荐作为第一版

```text
Code BRAM
    主要服务 I-side，D-side 可只读

Data BRAM
    主要服务 D-side，I-side 可选
```

优点：

- Harvard 并行访问；
- IFU 和 LSU 常态下不争用；
- 地址和权限清楚；
- Cache miss 更容易并行；
- RTL 和验证边界明确。

### 11.2 True Dual-Port BRAM

一块 BRAM 的两个端口都可以独立读写。

可以设计：

```text
Port A ← I-side
Port B ← D-side/Debug
```

但必须定义：

- 两端口同时写同一地址怎么办；
- 一端写、另一端读同一地址返回旧值还是新值；
- 两端时钟是否相同；
- Cache 与 Debug 写入的一致性；
- 是否绕过 Crossbar。

双端口不是“不需要仲裁”。同地址冲突仍需系统层定义。

### 11.3 一个统一 BRAM 的代价

如果把程序和数据都放进同一物理 BRAM：

- 链接和加载较灵活；
- `.text/.rodata/.data` 地址统一；
- 可以执行 RAM 中程序。

但：

- IFU 和 LSU 可能争用；
- Cache 一致性问题更多；
- 端口冲突要处理；
- Debug/DMA 加入后端口不足；
- Crossbar/Arbiter 更关键。

对当前比赛阶段，分离 Code BRAM 和 Data BRAM 更稳妥。

---

## 12. Cache 到底放在哪里

### 12.1 逻辑位置

最常见的位置是：

```text
CPU Pipeline
    ↓ CPU 内部请求接口
L1 Cache
    ↓ Cache Miss/Write 请求接口
SoC Interconnect
    ↓
Backing Memory
```

也就是：

```text
IFU → I-cache → HXI-I Master
LSU → D-cache → HXI-D Master
```

Cache 命中时，请求不会进入 HXI；只有：

- I-cache miss；
- D-cache load miss；
- write-through store；
- write-back eviction；
- uncached/MMIO；

才会进入 SoC 总线。

### 12.2 Cache 是不是“核里面的代码”

从 RTL 源码组织看，可以说是 CPU 核的一部分。但最好进一步区分：

```text
cpu_pipeline
    真正执行指令的流水线、寄存器、调度、提交

cpu_subsystem
    cpu_pipeline + I-cache + D-cache + 总线适配
```

推荐层次：

```text
cpu_subsystem.sv
├─ core_pipeline.sv
├─ icache.sv
├─ dcache.sv
├─ cache_control.sv
├─ hxi_i_master_adapter.sv
└─ hxi_d_master_adapter.sv
```

这样有两个好处：

1. 从 SoC 视角，CPU 只有两个规范的 HXI Master 接口；
2. 可以单独替换 Cache，而不修改执行流水线。

### 12.3 Cache 的存储阵列本身也可能使用 BRAM

Cache RTL 中通常有：

```text
Tag Array
Valid/Dirty Bits
Data Array
Replacement State
```

在 FPGA 上：

- 较大的 Data Array 通常推断为 BRAM；
- Tag Array 也可能推断为 BRAM；
- 较小的 Valid/Dirty 位可使用寄存器或 LUTRAM。

因此系统里可能同时存在：

```text
L1 Cache BRAM：保存主存数据的副本
Code/Data BRAM：作为真实 backing memory
```

它们不是同一块逻辑存储器。

---

## 13. 当前工程中的 Cache 实际情况

截至当前 `dev-v4.0`：

### 13.1 取指路径

```text
Frontend/IFU
    ↓ irom_addr/irom_ena
IROM_0
    ↓ irom_data
Frontend
```

取指仍直接连接 IROM，没有独立 `icache.sv`。

### 13.2 数据路径

```text
LSU / Load Queue / Store Buffer
    ↓
dcache
    ↓
dmem_regslice
    ↓ req_valid/ready + response
SocMemBridge
    ├─ DramBramAdapter → DRAM_0 BRAM
    └─ 简单 MMIO
```

现有 `dcache`：

- 位于 `core_top` 内；
- Direct-Mapped；
- Cache Line 为 16 Byte；
- `DCACHE_LINES=2048`，数据容量约 32 KiB；
- 一条 Line 使用四个 32 位 word bank；
- Load miss 用四次 word refill；
- Store 发送到底层存储，同时命中时更新 Cache，属于偏 Write-Through 的简单策略；
- MMIO/uncached 请求绕过缓存；
- Tag 和 Data Bank 带 `ram_style="block"`，目标是映射到 FPGA BRAM；
- Cache reset 通过初始化状态逐项清 Valid，而不是复位整个 Data Array。

### 13.3 现有 SoC 还不是 HXI Crossbar

当前 `SocMemBridge` 只有一条来自 D-side 的请求通道：

```text
一个 Master
    ↓ 地址比较
DRAM 或几个 MMIO 寄存器
```

它具有 request/response 握手的雏形，但还不具备：

- I-side 作为第二个 Master；
- 多个可复用 Slave Port；
- 每个 Slave 的独立仲裁；
- 不同 Slave 并行；
- APB 外设子系统；
- Default Error Slave；
- 正式的响应 owner 跟踪；
- 可扩展的连接矩阵。

因此新架构不是推倒 Cache 重写，而是：

```text
保留并整理 CPU 内部 D-cache
把 I-side 也规范化为 Master
把 SocMemBridge 演进为 HXI Crossbar + Slave Adapters
```

---

## 14. BRAM 已经很快，还需要 Cache 吗

这个问题不能只回答“需要”或“不需要”。

### 14.1 I-cache 对专用 Code BRAM 的收益

如果 Code BRAM：

- 专门供 IFU 使用；
- 固定 1 周期返回；
- 没有总线仲裁；
- CPU 每次只取一个 32 位指令；

I-cache 命中也可能需要 1 周期 BRAM 读取。此时 I-cache 不一定更快，反而增加：

- Tag 比较；
- Miss FSM；
- BRAM 消耗；
- reset/invalidate；
- `FENCE.I` 处理。

在这种情况下，小型预取队列或一次读取 64/128 位指令块，可能比完整 I-cache 更划算。

### 14.2 什么时候 I-cache 有价值

如果：

- Code Memory 经 Crossbar；
- Code BRAM 需要 2 周期；
- IFU 与 D-side/Debug 争用；
- 未来代码在 DDR/Flash；
- 一次可按 Cache Line 宽度读取；
- 分支预测和 Fetch Queue 需要更高带宽；

I-cache 就有明显价值。

### 14.3 D-cache 对 Data BRAM 的收益

Data BRAM 原生可能只需 1 周期，但实际路径还包括：

```text
LSU
→ Store/Load Queue
→ Crossbar
→ BRAM Adapter
→ BRAM
→ Response Route
→ LSU
```

总延迟可能达到 2～4 周期。D-cache 命中可以在 CPU 子系统内完成，因此仍可能提升 CoreMark。

但如果 D-cache 本身使用同步 BRAM，命中也至少需要一次阵列读取。收益要通过：

- hit latency；
- miss rate；
- refill 开销；
- BRAM 资源；
- 时序频率；

综合判断。

### 14.4 当前 32 KiB D-cache 是否合适

当前 Data BRAM 窗口约 256 KiB，而 D-cache 数据容量约 32 KiB，再加 Tag BRAM。

对 FPGA 比赛可以工作，但建议进行对比：

```text
No D-cache
4 KiB Direct-Mapped
8 KiB Direct-Mapped
16 KiB Direct-Mapped
32 KiB Direct-Mapped
```

比较：

- CoreMark；
- FPGA BRAM 使用量；
- 最高频率；
- hit rate；
- miss penalty；
- `stall_mem`；
- 功耗或资源。

Cache 越大不一定分数越高。大 Tag/Index 可能降低时序，额外 BRAM 也可能挤占 Code/Data RAM。

### 14.5 对当前阶段的建议

```text
I-side：
先保留专用 Code BRAM + Fetch Queue
如果取指延迟/争用成为瓶颈，再增加 4～8 KiB I-cache

D-side：
保留现有 D-cache 功能
先验证 4～16 KiB 规模与当前 32 KiB 的性价比
第一版继续采用 Write-Through，降低一致性和精确异常复杂度
```

---

## 15. Cache 与 MMIO 的边界

UART、GPIO、Timer、PLIC 等 MMIO 必须：

```text
Non-cacheable
不可预取
不可合并
不可投机产生副作用
按规定顺序完成
```

### 15.1 地址属性应由 SoC 规范决定

建议在 CPU 子系统或地址属性模块中定义：

```text
Code BRAM：cacheable、executable
Data BRAM：cacheable，可按需要 executable
APB Window：non-cacheable、non-executable、I/O
CLINT：non-cacheable、non-executable、I/O
Interrupt Controller：non-cacheable、non-executable、I/O
```

不要只依赖软件随意提供 `uncached` 信号。硬件至少应根据地址范围强制 MMIO uncached。

### 15.2 StoreBuffer 也不能把 MMIO 当普通 RAM

普通 RAM Store 可以：

- 暂存在 StoreBuffer；
- 合并；
- 延后写出。

MMIO Store 可能触发：

- UART 发送；
- 中断清除；
- Timer 重载；
- GPIO 翻转；

因此必须在精确提交边界执行，不能被错误丢弃、重复或越过更早的访问。

---

## 16. I-cache、D-cache 与一致性

### 16.1 单核 RT-Thread 不需要每次切线程都清 Cache

当前目标是：

- 单核；
- M-mode；
- 无 MMU；
- 所有线程共享同一物理地址空间。

线程切换只保存寄存器上下文，不需要清空 I-cache/D-cache。

### 16.2 什么时候需要 `FENCE.I`

如果软件通过 D-side 写入一段将要执行的代码：

```text
Bootloader 写 Data/Code RAM
调试器下载代码
运行时修改指令
```

I-cache 或取指队列可能仍保留旧指令。

RISC-V 使用：

```text
FENCE.I
```

同步数据写入与后续取指。

简单实现可以在执行 `FENCE.I` 时：

- 等待 StoreBuffer 清空；
- 等待 D-side 未完成写结束；
- 使 I-cache 全部 Invalid；
- 清空 Fetch Queue/指令缓冲；
- 从下一条 PC 重新取指。

如果 Code BRAM 运行时只读，也没有 DMA/Debug 修改代码，问题会简单很多。

### 16.3 DMA 带来的 D-cache 问题

未来增加 DMA 后：

```text
CPU D-cache 中有新数据
但尚未写回主存
DMA 从主存读取旧数据
```

或：

```text
DMA 更新主存
CPU D-cache 仍命中旧副本
```

Write-Through Cache 可以减轻第一类问题，但第二类仍需：

- DMA 一致性硬件；
- 软件 invalidate；
- DMA 使用 non-cacheable buffer；
- 明确 Cache maintenance 接口。

因此第一版预留 Debug/DMA Master 可以，但不要在没有一致性方案时让 DMA 随意访问 Cacheable RAM。

---

## 17. 推荐的完整 SoC 结构

```text
fpga_top
├─ Clock Wizard / PLL
├─ Reset Synchronizer
├─ UART pins
├─ GPIO pins
└─ soc_top
   │
   ├─ cpu_subsystem
   │  ├─ core_pipeline
   │  ├─ frontend / fetch_queue / predictor
   │  ├─ optional icache
   │  ├─ LSU / load_queue / store_buffer
   │  ├─ dcache
   │  ├─ HXI-I Master
   │  ├─ HXI-D Master
   │  └─ IRQ inputs
   │
   ├─ hxi_crossbar
   │  ├─ address_decoder
   │  ├─ per_slave_arbiter
   │  ├─ request_router
   │  ├─ response_router
   │  └─ owner_state
   │
   ├─ memory_subsystem
   │  ├─ hxi_code_bram_slave
   │  │  └─ code_bram
   │  └─ hxi_data_bram_slave
   │     └─ data_bram
   │
   ├─ hxi_to_apb_bridge
   │  └─ apb_interconnect
   │     ├─ apb_uart
   │     ├─ apb_gpio
   │     ├─ apb_perf_ctrl
   │     └─ reserved
   │
   ├─ machine_timer_or_clint
   ├─ interrupt_controller
   └─ default_error_slave
```

### 17.1 为什么 CLINT/Timer 可以直接挂 HXI

Machine Timer 可能被：

- RT-Thread Tick 高频访问；
- 64 位拆分读写；
- 中断处理程序频繁更新。

直接挂 HXI 可以减少 APB 两阶段延迟。

但如果频率不高，也可以放在 APB。关键是统一规范，而不是必须采用某一种。

### 17.2 UART/GPIO 为什么放 APB

它们的访问特点是：

- 以寄存器为主；
- 带宽很低；
- 时序不紧；
- 外设数量容易增加。

放到 APB 后，每个外设只实现简单寄存器接口，Crossbar 的 Slave 数也不会随外设数量快速增加。

---

## 18. 一份暂定地址规划

以下只是一份与当前工程连续演进的建议，不是已经冻结的最终地址。

| 地址范围 | 暂定用途 | 属性 |
| --- | --- | --- |
| `0x8000_0000...` | Code BRAM | 可执行、可缓存，D-side 至少可读 |
| `0x8010_0000–0x8013_FFFF` | Data BRAM，沿用当前 256 KiB 窗口 | 读写、可缓存 |
| `0x8020_0000...` | APB Peripheral Window | 不缓存、不可执行 |
| 单独预留窗口 | CLINT/Machine Timer | 不缓存、不可执行 |
| 单独预留窗口 | Interrupt Controller | 不缓存、不可执行 |
| 其余 | Default Slave | Access Fault |

Code BRAM 容量应由 RT-Thread + FinSH + CoreMark 的实际 ELF 决定。当前取指地址只使用 `irom_addr[13:2]`，等价于约 16 KiB 指令空间，对完整系统明显偏小。

建议先用链接结果确定：

```text
128 KiB
256 KiB
或更大
```

而不是先拍脑袋固定。

---

## 19. 推荐的 RTL 文件组织

```text
rtl/
├─ top/
│  ├─ fpga_top.sv
│  └─ soc_top.sv
│
├─ cpu/
│  ├─ cpu_subsystem.sv
│  ├─ core_pipeline.sv
│  ├─ frontend/
│  ├─ execute/
│  ├─ memory/
│  └─ csr_trap/
│
├─ cache/
│  ├─ icache.sv
│  ├─ dcache.sv
│  ├─ cache_tag_bank.sv
│  └─ cache_data_bank.sv
│
├─ bus/
│  ├─ hxi_pkg.sv
│  ├─ hxi_crossbar.sv
│  ├─ hxi_decoder.sv
│  ├─ hxi_arbiter.sv
│  ├─ hxi_req_router.sv
│  ├─ hxi_rsp_router.sv
│  ├─ hxi_default_slave.sv
│  └─ hxi_to_apb_bridge.sv
│
├─ memory/
│  ├─ hxi_bram_slave.sv
│  ├─ code_bram.sv
│  ├─ data_bram.sv
│  └─ memory_map_pkg.sv
│
├─ peripheral/
│  ├─ apb_interconnect.sv
│  ├─ apb_uart.sv
│  ├─ apb_gpio.sv
│  ├─ machine_timer.sv
│  └─ interrupt_controller.sv
│
└─ clk_rst/
   ├─ reset_sync.sv
   └─ clock_control.sv
```

建议把地址常量集中在：

```text
memory_map_pkg.sv
```

不要在 Crossbar、Cache、BSP 和外设中各自复制一套 magic number。

---

## 20. 建议的实施顺序

### 阶段 1：先冻结 HXI 协议

完成：

- 请求字段；
- 响应字段；
- valid/ready 保持规则；
- 一个 outstanding 限制；
- 写完成语义；
- 错误响应；
- 非对齐行为；
- Cacheable/MMIO 属性；
- 仲裁规则。

在协议没有冻结前，不要同时编写很多 Slave。

### 阶段 2：单 Master、两个 Slave

先做：

```text
一个测试 Master
    ↓
HXI Crossbar
    ├─ Code/Data BRAM Model
    └─ Default Slave
```

验证译码、背压和错误。

### 阶段 3：扩展到 IFU + LSU 两个 Master

验证：

- 不同 Slave 并行；
- 同一 Slave 仲裁；
- 响应不会串给另一 Master；
- 一个 Master 背压不阻塞无关路径。

### 阶段 4：接入真实 BRAM Adapter

先使用行为模型，再替换成：

- Vivado BRAM inference；
- 或 Block Memory Generator IP。

检查 1/2 周期读延迟和 byte write enable。

### 阶段 5：接 HXI-to-APB

只先挂：

```text
UART
GPIO
```

再增加 Timer 和中断控制器。

### 阶段 6：接 CPU，不开 Cache

让：

```text
IFU → HXI-I
LSU → HXI-D
```

先跑：

- 取指；
- Load/Store；
- UART；
- ISA 测试；
- 中断。

### 阶段 7：接 D-cache

验证：

- hit；
- miss；
- refill；
- store；
- uncached MMIO；
- flush/kill；
- StoreBuffer 顺序。

### 阶段 8：决定是否加入 I-cache

根据：

- 取指 stall；
- Code BRAM 延迟；
- Crossbar 争用；
- BRAM 剩余量；
- CoreMark；

再做决定。

---

## 21. 验证计划

### 21.1 HXI 协议断言

至少检查：

```text
valid && !ready 时 payload 保持
rsp_valid && !rsp_ready 时 response 保持
一次请求只被一个 Slave 接收
一次响应只返回一个 Master
没有请求不能产生幽灵响应
一个 outstanding 模式下不能重复发请求
```

### 21.2 Crossbar 测试

```text
M0→S0，M1→S1：同周期接受
M0→S0，M1→S0：正确仲裁
Slave 长时间 backpressure
Master 长时间不接 response
未映射地址返回 error
连续读写不同 Slave
仲裁轮转不饥饿
```

### 21.3 BRAM Adapter 测试

```text
word read/write
byte lane 0001/0010/0100/1000
halfword lane
所有 wstrb 组合
读延迟
输出寄存器开关
边界地址
非法地址
read-during-write
双端口同地址冲突
```

### 21.4 Cache 测试

```text
Cold miss
Hit
同 Index 不同 Tag 的 conflict miss
Line refill 四个 word
Store hit
Store miss
MMIO bypass
reset invalidate
branch flush 时未完成请求
FENCE.I
Cacheable 边界
```

### 21.5 系统级测试

```text
IFU 连续取指，同时 LSU 访问 Data BRAM
IFU 与 LSU 同时访问同一个 BRAM
RT-Thread Tick 中断期间 Cache miss
UART 中断与 CoreMark 并行
非法 MMIO 地址触发 access fault
CoreMark 前后统计 cache hit/miss
```

---

## 22. HXI、AHB、AXI-Lite 三种选择怎样取舍

| 方案 | 优点 | 代价 | 当前项目建议 |
| --- | --- | --- | --- |
| 自定义 HXI | 与当前接口接近、字段少、容易针对 Cache/精确异常设计 | 必须自己写完整规范和验证，不能直接称标准兼容 | **推荐** |
| AHB-Lite + Matrix | 标准成熟、适合 MCU、很多 SRAM/APB 参考 | CPU/Cache 接口要改，AHB 相位规则需要重新学习和验证 | 若计划复用 AHB IP，可选 |
| AXI4-Lite | valid/ready，生态丰富，Vivado IP 方便 | 五通道，写地址/写数据解耦，桥和验证复杂 | 当前没有必要 |
| TileLink-UL | RISC-V 开源生态中常见，请求/响应清晰 | 字段和语义比当前 HXI 多，现有 IP 不直接兼容 | 仅在接入 Chipyard/Rocket IP 时考虑 |

HXI 的关键不是名字，而是：

```text
协议完整
响应不串线
背压正确
错误可结束
MMIO 不缓存
Cache miss 可恢复
RTL 可断言验证
```

---

## 23. 最终建议方案

结合你的 RTL 背景、当前 D-cache、比赛目标和优秀作品，建议采用：

### 主干

```text
2×N HXI Crossbar
```

初始两个 Master：

```text
I-side
D-side
```

预留第三个：

```text
Debug/DMA
```

### 存储

```text
Code BRAM：独立物理 BRAM，预初始化，D-side 至少可读
Data BRAM：独立物理 BRAM，支持四字节写掩码
```

两者都通过 `hxi_bram_slave` 连接，不把 Xilinx IP 端口暴露到 Crossbar。

### Cache

```text
Cache 位于 CPU Pipeline 与 HXI 之间
D-cache 保留，优先采用简单 Write-Through
I-cache 根据取指瓶颈决定，接口先预留
MMIO 永远 bypass
```

从源码层次上将 Cache 放在 `cpu_subsystem` 中，而不是和 UART/BRAM 一起放到 SoC 外设目录。

### 外设

```text
HXI-to-APB Bridge
    ↓
UART/GPIO/低速寄存器外设
```

Timer/CLINT 和 Interrupt Controller 可根据访问频率选择直接 HXI 或 APB，但地址和中断语义必须统一。

### 协议复杂度

第一版：

```text
一个 outstanding
单拍
顺序响应
Round-Robin 仲裁
显式错误响应
```

等 RT-Thread 和 CoreMark 稳定后，再考虑 burst、多个 outstanding、DMA 和 Cache 一致性。

---

## 24. 你现在需要真正定下来的设计问题

进入 RTL 修改前，应形成一页明确决策：

1. Code BRAM 和 Data BRAM 各多大；
2. `.rodata` 放在哪里；
3. D-side 是否可以读取/写入 Code BRAM；
4. 是否支持从 Data BRAM 执行；
5. BRAM 原生读延迟是 1 还是 2 周期；
6. HXI 写请求何时算完成；
7. 每个 Master 是否只允许一个 outstanding；
8. 同一 Slave 的仲裁策略；
9. Default Slave 怎样返回错误；
10. 哪些地址是 Cacheable、Executable、MMIO；
11. D-cache 容量是否继续使用 32 KiB；
12. D-cache 是 Write-Through 还是 Write-Back；
13. 是否第一版就增加 I-cache；
14. `FENCE` 和 `FENCE.I` 在 CPU 内如何完成；
15. 外设是否统一放入 APB 子系统；
16. Debug/UART 下载是否需要写 Code BRAM。

这些决定会同时影响：

- CPU 顶层端口；
- Crossbar；
- Cache；
- BRAM IP；
- 链接脚本；
- `start.S`；
- RT-Thread BSP；
- 仿真 Testbench；
- Vivado 初始化流程。

---

## 25. 参考资料

- 去年优秀作品：《CICC0905422 全国总决赛技术文档》，重点参考第 2.4 节 HXI 总线、ROM/RAM/外设结构和 BRAM 读延迟说明；
- [Arm：Introduction to AMBA AXI4](https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/102202_0100_01_Introduction_to_AMBA_AXI.pdf)，其中包含 AHB、AHB-Lite、APB 的定位和历史关系；
- [Arm Cortex-M4 产品说明](https://developer.arm.com/compute-ip/cortex-m4)，可观察 Harvard CPU 使用多个 AHB-Lite 接口的实例；
- [Arm CMSDK 介绍](https://developer.arm.com/community/arm-community-blogs/b/architectures-and-processors-blog/posts/10-useful-facts-of-the-cortex-m0-system-design-kit-cmsdk)，包含 AHB Bus Matrix、AHB-to-SRAM 和 APB 外设组织方式；
- [AMD 7 Series FPGA Memory Resources UG473](https://docs.amd.com/api/khub/documents/9gZGbqBxtlKXxBfkBt~lAg/content)，用于核对 Kintex-7 BRAM 的同步读写和端口行为；
- [AMD Block Memory Generator PG058](https://docs.amd.com/api/khub/documents/Uso4QLBXXtlx2fgym9Z7Vw/content)，用于了解 COE 初始化、端口类型和字节写使能；
- [RISC-V 非特权指令集规范](https://docs.riscv.org/reference/isa/v20240411/_attachments/riscv-unpriv.pdf)，用于 `FENCE`、`FENCE.I` 与指令/数据流同步要求。
