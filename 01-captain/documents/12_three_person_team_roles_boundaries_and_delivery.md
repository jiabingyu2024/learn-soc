# 三人团队的职责边界、交付标准与协作方式

> 适用目标：自研 RV32 CPU + FPGA SoC + RT-Thread + FinSH + CoreMark。  
> 本文解决的主要问题不是“把任务平均分成三份”，而是避免最后所有设计、调试和收尾工作重新落到队长一个人身上。  
> 核心原则：每个人拥有一个完整责任域，并对该责任域的设计、实现、测试、文档、进度和缺陷修复负责。

---

## 0. 先说结论：过去的分工为什么没有真正成立

过去看起来已经把任务分给三个人，但实际很可能是下面这种模式：

```text
队长：
    知道总体架构
    知道最终需要跑什么
    知道模块之间怎样连接
    知道出问题时怎样定位

其他队员：
    只知道自己被分配了几个文件
    不知道输入来自哪里
    不知道输出交给谁
    不知道完成到什么程度才算结束
    代码写完后由队长集成和调试
```

这种模式下，表面分工是：

```text
队员写代码
队长做集成
```

实际分工却变成：

```text
队员提交半成品
队长理解半成品
队长修正接口
队长补测试
队长解决集成问题
队长负责最终结果
```

所以队员写得越多，队长需要重新理解和返工的内容可能越多，最后不如队长自己使用 AI 完成。

问题不一定是队员能力不足，而是缺少以下内容：

1. 统一的系统架构；
2. 明确的责任边界；
3. 固定的接口契约；
4. 可检查的交付物；
5. 缺陷归属规则；
6. 独立完成工作的权限和责任。

新的分工必须从：

```text
“你帮我写这几个文件”
```

改成：

```text
“这一整块功能由你负责，边界以内你决定、实现、测试和维护；
边界以外按接口使用，不能把集成后的问题全部交回队长。”
```

---

## 1. 三人团队采用三个完整责任域

推荐的总体划分是：

```text
队长：系统架构、CPU 核、接口契约和最终集成

队员 A：SoC RTL、互联、存储适配、外设和 FPGA 基础设施

队员 B：启动/BSP、RT-Thread、驱动、应用、构建和系统回归
```

完整关系：

```text
┌─────────────────────────────────────────────────────────┐
│ 队长：系统架构与最终集成                                │
│                                                         │
│  架构决策  CPU Core  CPU Wrapper  接口契约  Release     │
└──────────────┬───────────────────────┬──────────────────┘
               │ CPU ↔ SoC 契约        │ SoC ↔ 软件契约
               │                       │
┌──────────────▼─────────────┐  ┌──────▼──────────────────┐
│ 队员 A：SoC RTL            │  │ 队员 B：软件与系统验证 │
│                            │  │                         │
│ HXI / BRAM / APB / UART    │  │ start / link / trap    │
│ Timer / IRQ / Clock/Reset  │  │ RT-Thread / FinSH      │
│ 外设单元测试 / FPGA 实现   │  │ CoreMark / 构建 / 回归 │
└────────────────────────────┘  └─────────────────────────┘
```

这里不是说队长完全不懂 SoC RTL 或软件，也不是说队员不能查看其他目录。

区别在于：

```text
理解全局 ≠ 亲自实现全部
可以评审 ≠ 自动接管
可以协助定位 ≠ 替责任人修完
```

---

## 2. 什么叫“拥有一个责任域”

一个功能的 Owner 不只是负责写第一版代码。

Owner 必须同时负责：

```text
需求理解
    ↓
方案设计
    ↓
接口确认
    ↓
代码实现
    ↓
单元测试
    ↓
集成验证
    ↓
问题定位
    ↓
自己边界内的缺陷修复
    ↓
文档和使用说明
```

如果队员 A 负责 Timer，那么他的工作不是：

```text
写一个 machine_timer.sv，然后发给队长
```

而应该是：

```text
明确 Timer 寄存器和频率
实现 mtime/mtimecmp/IRQ
实现 HXI 访问
写单元测试
验证 RV32 高低 32 位读写
提供 BSP 所需寄存器表
接入 SoC
发现 Timer 边界问题时自己修复
```

如果队员 B 负责 RT-Thread BSP，那么他的工作不是：

```text
把 RT-Thread 源码复制到工程
```

而应该是：

```text
完成启动、链接、Trap、上下文切换
完成 UART/Timer 驱动
完成 rtconfig
让线程、延时、FinSH、CoreMark 真正运行
提供构建命令和日志
遇到软件边界内问题时自己定位修复
```

---

## 3. 三个人都必须理解的最小全局架构

不能要求每个人都深入掌握全部实现，但三个人必须能讲清楚以下五条路径。

### 3.1 启动路径

```text
FPGA 上电
→ PLL locked
→ 释放 CPU 复位
→ PC = Reset Vector
→ Code BRAM 返回 start.S
→ 初始化 sp / .data / .bss
→ RT-Thread 初始化
→ main 线程
→ FinSH
```

### 3.2 普通访存路径

```text
CPU LSU
→ D-cache
→ HXI-D
→ Crossbar
→ Data BRAM Adapter
→ Data BRAM
→ Response
→ CPU
```

### 3.3 MMIO 路径

```text
CPU Store/Load
→ 识别为 non-cacheable
→ HXI Crossbar
→ Timer 或 HXI-to-APB
→ UART/GPIO 寄存器
```

### 3.4 Timer 中断路径

```text
mtime >= mtimecmp
→ irq_timer
→ mip.MTIP
→ CPU 精确进入 mtvec
→ Trap 保存上下文
→ rt_tick_increase()
→ 可能调度线程
→ mret
```

### 3.5 编译和镜像路径

```text
C/汇编源码
→ RISC-V 交叉编译器
→ ELF
→ map/dump
→ BIN/HEX/COE/MEM
→ FPGA BRAM
→ CPU 从 Reset Vector 执行
```

三个人都要知道：

- 哪个地址放代码；
- 哪个地址放数据；
- UART 和 Timer 在哪里；
- 中断怎样进入 CPU；
- 出错时先看日志、反汇编还是波形。

但不要求三个人都能独立重写 CPU、Crossbar 和 RT-Thread。

---

## 4. 队长的完整责任域

队长的定位是：

```text
System Architect
+ CPU Owner
+ Integration Owner
+ Release Owner
```

队长承担更多系统责任，但不能变成“所有没人完成的工作都由队长接管”。

### 4.1 队长负责的设计内容

#### 系统架构

- 冻结总体 SoC 层次；
- 决定 CPU、Cache、HXI、BRAM、APB 和外设怎样连接；
- 冻结第一版功能范围；
- 控制新增功能，防止项目无限膨胀；
- 维护系统方框图和启动路径。

#### 硬件—软件契约

- Memory Map；
- HXI 主接口语义；
- CPU 中断端口；
- CSR 和 Trap 语义；
- Cacheable/Uncached 属性；
- Reset Vector；
- 时钟频率和 Timer timebase；
- 错误响应与异常映射。

#### CPU Core

- CPU 流水线和提交；
- CSR；
- 同步异常；
- 异步中断；
- `mret`；
- 精确 Trap；
- Cache 与 Store Buffer 的 CPU 侧语义；
- CPU 对 HXI 请求/响应的行为；
- CPU 回归测试。

#### 最终集成

- 维护 `soc_top/student_top` 级连接；
- 确认 CPU、SoC 和 BSP 三方接口版本一致；
- 控制主分支集成顺序；
- 决定一个版本是否可以成为里程碑；
- 组织最终 FPGA 镜像和比赛版本；
- 负责最终答辩的系统主线。

### 4.2 队长应拥有的文件

建议由队长直接负责：

```text
rtl/core/**
rtl/cpu/**                    # 若后续建立 cpu_subsystem 目录
rtl/top/soc_top.sv            # 或当前 student_top.sv 的最终集成
docs/architecture/**
docs/interfaces/**
docs/memory_map.md
docs/interrupt_map.md
docs/clock_reset.md
scripts/filelists/top-level 部分
release/**
```

模块 Owner 可以修改自己的 filelist 条目，但最终顶层 filelist 和 Release 配置由队长批准。

### 4.3 队长必须交付什么

#### 架构包

至少包括：

```text
1. system_block_diagram.md
2. memory_map.md
3. hxi_interface.md
4. cpu_soc_interface.md
5. interrupt_map.md
6. clock_reset.md
7. boot_flow.md
8. milestone_plan.md
```

这些文档应该短而确定，不是几十页学习资料。队员工作时以架构包为准。

#### CPU 可验证版本

- ISA 回归通过；
- 同步异常通过；
- Timer IRQ 注入测试通过；
- `mcause/mepc/mstatus/mret` 通过；
- MMIO 请求能够被观察；
- 非法总线响应能够转换为 Access Fault；
- 提供可供 SoC 使用的稳定 CPU 顶层。

#### 集成版本

每个里程碑应留下：

- Git 提交号；
- 构建命令；
- 使用的 RT-Thread/CoreMark 版本；
- ELF/bitstream 名称；
- 通过的测试；
- 已知问题。

### 4.4 队长不应该做什么

以下情况不能默认由队长解决：

- 队员 A 的 UART 状态机有 Bug，队长直接重写；
- 队员 A 的 Crossbar 串响应，队长长期接管调试；
- 队员 B 的链接脚本错误，队长直接修完；
- 队员 B 不会使用 RT-Thread SCons，队长替他维护构建；
- 队员提交没有测试的 AI 代码，队长负责理解和验收全部细节；
- 因为队员进度慢，队长悄悄把任务做完。

队长可以：

- 解释架构；
- 指出最早失败的接口；
- 提供 CPU 侧或集成侧证据；
- 评审方案；
- 决定契约；

但边界内的修复仍由对应 Owner 完成。

### 4.5 队长的完成标准

队长不是以“自己写了多少代码”衡量，而是以：

```text
系统契约是否明确
CPU 是否稳定
两个队员能否独立工作
集成问题能否快速归属
主分支是否持续可运行
最终版本是否可重复构建
```

衡量。

---

## 5. 队员 A：SoC RTL Owner

队员 A 的定位是：

```text
SoC Infrastructure Owner
+ Peripheral RTL Owner
+ FPGA Hardware Owner
```

这是一块完整的硬件责任域，不是“帮队长写外设”。

### 5.1 队员 A 负责的模块

#### HXI 互联

- HXI package/interface；
- 地址译码；
- Crossbar；
- 每 Slave 仲裁；
- Request Router；
- Response Router；
- Owner 记录；
- Default Error Slave；
- HXI 协议断言。

#### 存储子系统

- Code BRAM Adapter；
- Data BRAM Adapter；
- BRAM IP/推断 RAM 封装；
- 同步读延迟；
- 字节写掩码；
- 初始化接口；
- 双口冲突规则；
- 仿真模型与 FPGA IP 一致性。

#### 外设子系统

- HXI-to-APB Bridge；
- APB Interconnect；
- CPU 可访问的 UART；
- GPIO；
- Machine Timer/CLINT-MTIMER；
- 简化外部中断控制器；
- 原有 Counter 的保留或整理。

#### 时钟、复位和 CDC

- 各时钟域分组；
- Reset Synchronizer；
- UART/CPU 域 CDC；
- Timer timebase；
- 外设 IRQ 同步；
- FPGA PLL/Clock Wizard 接入；
- XDC 中与自己模块相关的引脚和时钟约束信息。

### 5.2 队员 A 应拥有的文件

推荐：

```text
rtl/bus/**
rtl/memory/**
rtl/peripheral/**
rtl/clk_rst/**
rtl/ip_wrapper/**
rtl/soc/SocMemBridge.sv       # 过渡期
rtl/soc/DramBramAdapter.sv
rtl/soc/counter.sv
rtl/soc/uart.sv
rtl/soc/clock/reset 相关文件
tb/unit/hxi/**
tb/unit/memory/**
tb/unit/peripheral/**
docs/peripheral/**
docs/register_map/**
fpga/ip/**
fpga/constraints/外设部分
```

`soc_top/student_top.sv` 最终所有权属于队长，但队员 A 必须提供可直接实例化的 SoC 模块和明确端口。

### 5.3 队员 A 的输入

队长必须提供：

- HXI 协议；
- Master 数量；
- Slave 列表；
- Memory Map；
- CPU IRQ 端口；
- 时钟频率；
- 复位极性；
- BRAM 容量；
- Timer timebase；
- 第一版是否需要 APB/PLIC。

队员 B 必须提供：

- UART 驱动需要的寄存器功能；
- Timer 驱动需要的寄存器功能；
- RT-Thread Console 的收发需求；
- 中断清除方式是否方便驱动实现；
- CoreMark 计时接口需求。

### 5.4 队员 A 的输出

不能只输出 RTL 文件。必须输出：

#### 可实例化模块

```text
hxi_crossbar
hxi_bram_slave
machine_timer
hxi_to_apb_bridge
apb_uart
apb_gpio
interrupt_controller
```

#### 软件可见寄存器表

每个外设写清：

```text
Base Address
Offset
Width
RO/RW/W1C
Reset Value
Bit Meaning
Side Effect
IRQ Condition
IRQ Clear Method
```

#### 单元测试证据

例如 Timer：

- 计数频率正确；
- `mtimecmp` 比较正确；
- IRQ 是电平；
- RV32 高低半部读写通过；
- 非法地址返回 error。

例如 UART：

- TX 字节序列正确；
- RX 采样正确；
- FIFO 满/空正确；
- RX IRQ 和清除正确；
- APB/HXI 背压正确。

例如 Crossbar：

- 不同 Slave 并行；
- 同一 Slave 正确仲裁；
- Response 返回正确 Master；
- Default Slave 返回错误；
- 背压时 payload 保持。

#### BSP 使用说明

提供：

```text
soc_memory_map.h 所需常量
寄存器访问顺序
Timer 频率
UART 波特率计算方法
中断号和清除方法
```

### 5.5 队员 A 不能自行修改什么

未经队长批准，不得自行修改：

- CPU 顶层端口；
- HXI 字段和握手语义；
- Memory Map；
- Reset Vector；
- CPU CSR；
- `mcause` 编码；
- Cacheable/Uncached 划分；
- 软件 ABI；
- RT-Thread 内核源码。

如果现有契约确实无法实现，队员 A 应提出接口变更请求，而不是直接改 CPU 或 BSP。

### 5.6 队员 A 遇到问题时的责任

如果系统表现为：

```text
CPU 已正确发出一笔符合 HXI 契约的 UART 写
但 UART 没有发送
```

队员 A 负责从：

```text
HXI Slave 接收
→ Bridge
→ APB
→ UART FIFO
→ TX 引脚
```

完整定位和修复。

不能只回复：

```text
“单独仿真好像能跑，可能是队长集成的问题。”
```

Owner 必须提供边界波形，证明最早失败点是否在自己范围内。

### 5.7 队员 A 的完成标准

SoC RTL 责任域完成必须满足：

- 模块独立测试通过；
- 顶层接口与架构包一致；
- 软件可见寄存器有文档；
- Verilator 行为模型可用；
- FPGA 综合使用真实 BRAM/IP；
- 非法访问不会死锁；
- Timer、UART IRQ 可在 CPU 边界观察；
- 发现自己模块缺陷时由自己修复；
- 队员 B 可以只根据寄存器文档写驱动。

---

## 6. 队员 B：BSP、RT-Thread 与系统验证 Owner

队员 B 的定位是：

```text
Firmware/BSP Owner
+ RT-Thread Port Owner
+ Application Integration Owner
+ End-to-End Regression Owner
```

这里的“系统验证 Owner”是负责运行和维护端到端测试，不表示 CPU/SoC 的 Bug 也由他修。

### 6.1 队员 B 负责的软件内容

#### 启动和链接

- `start.S`；
- 栈初始化；
- `.data` 搬运；
- `.bss` 清零；
- `gp`；
- Trap 向量安装；
- `link.lds`；
- ELF section 检查；
- map/dump 检查；
- 镜像尺寸检查。

#### RISC-V Port

- Trap 汇编入口；
- 上下文保存和恢复；
- 中断栈；
- `rt_hw_context_switch`；
- `rt_hw_context_switch_to`；
- `rt_hw_context_switch_interrupt`；
- `mstatus/mie/mtvec/mscratch` 使用；
- `mret` 返回；
- 中断 enter/leave。

#### BSP

- `board.c`；
- UART Console 驱动；
- Timer Tick 驱动；
- 中断分发；
- `rt_hw_tick_init()`；
- heap 边界；
- 时钟频率宏；
- `rtconfig.h`；
- `SConscript`/Makefile。

#### RT-Thread 和应用

- 最小 RT-Thread 配置；
- 线程演示；
- `rt_thread_delay()`；
- FinSH/MSH；
- CoreMark 命令包装；
- CoreMark `portme`；
- 现场版本 CoreMark 修改范围控制；
- 系统启动日志。

#### 构建和镜像

- 交叉编译工具链；
- SCons/Makefile 入口；
- ELF/BIN/HEX/COE/MEM 转换；
- 反汇编和 map 自动生成；
- BRAM 镜像更新；
- 可重复构建脚本；
- Release 软件包。

### 6.2 队员 B 应拥有的文件

推荐：

```text
software/startup/**
software/bsp/**
software/rt-thread/**
software/applications/**
software/coremark/**
software/linker/**
software/include/soc_memory_map.h
scripts/build_software/**
scripts/elf2mem/**
tests/baremetal/**
tests/rtthread/**
tests/system/**
docs/software/**
docs/test_reports/**
```

如果当前工程还没有这样的目录，应逐步建立，不要把软件文件散放在 RTL 目录中。

### 6.3 队员 B 的输入

队长必须提供：

- Reset Vector；
- Code/Data RAM 范围；
- CPU 支持的 ISA/ABI；
- CSR 和 Trap 语义；
- CPU IRQ cause；
- HXI 错误如何映射异常；
- 中断嵌套策略；
- Cache/FENCE 能力。

队员 A 必须提供：

- UART/Timer/IRQ 寄存器表；
- Memory Map 常量；
- Timer timebase；
- 中断号；
- 中断清除顺序；
- MMIO 对齐和写掩码规则；
- FPGA 镜像装载方式。

### 6.4 队员 B 的输出

#### 可重复构建

一条或少量固定命令完成：

```text
clean
compile
link
objdump
map
image conversion
```

不能依赖：

```text
手工复制文件
临时修改绝对路径
队长电脑上某个未知工具
只存在于聊天记录里的步骤
```

#### 裸机测试程序

至少提供：

```text
hello_uart
ram_test
timer_irq_test
trap_context_test
external_irq_test
```

#### RT-Thread 测试

至少验证：

- 内核启动；
- 两线程调度；
- 延时；
- 时间片；
- 软件定时器；
- UART Console；
- FinSH；
- CoreMark 命令。

#### 自动判定日志

例如：

```text
[BOOT] PASS
[RAM] PASS
[TIMER_IRQ] PASS count=1000
[CONTEXT] PASS
[RTTHREAD_SCHED] PASS
[FINSH] PASS
[COREMARK] PASS
```

#### 软件 Release 信息

- 编译器版本；
- 编译参数；
- RT-Thread commit/tag；
- CoreMark 原始版本；
- 被允许修改的文件；
- ELF 大小；
- `.text/.data/.bss/heap/stack` 使用量；
- 运行日志。

### 6.5 队员 B 不能自行修改什么

未经队长批准，不得：

- 为了让软件运行而修改 CPU CSR 行为；
- 自行更改 Memory Map；
- 在多个文件硬编码不同外设地址；
- 用轮询绕过应该验证的 Timer IRQ；
- 修改 CoreMark 算法主体；
- 为了通过测试删掉上下文寄存器保存；
- 把硬件 Bug 用固定延时或重复访问永久掩盖；
- 直接改队员 A 的 RTL 后不通知 Owner。

### 6.6 队员 B 遇到问题时的责任

如果：

```text
Timer IRQ 已正确进入 CPU
mcause/mepc 正确
Trap 入口也执行
但 RT-Thread Tick 不增加
```

这属于队员 B 的责任域：

```text
Trap 汇编
→ 中断分发
→ Timer ISR
→ rt_interrupt_enter
→ rt_tick_increase
→ rt_interrupt_leave
```

队员 B 必须自己定位，而不是把整个问题交给队长。

如果发现：

```text
软件正确写了 mtimecmp
但硬件 IRQ 没撤销
```

应提供：

- 写入地址；
- 写入值；
- 反汇编；
- HXI/APB 访问证据；
- 期望与实际；

然后交给队员 A 修复 Timer RTL。

### 6.7 队员 B 的完成标准

- 新电脑按文档可以完成构建；
- ELF 地址与链接脚本一致；
- map/dump 自动保存；
- 裸机 UART/Timer/Trap 测试通过；
- RT-Thread Tick 和上下文切换通过；
- FinSH 可交互；
- CoreMark 按比赛允许范围接入；
- 系统测试有自动或半自动判定；
- 软件问题由自己修复；
- 硬件问题能提供边界证据后再移交。

---

## 7. 三个关键边界怎样划分

### 7.1 边界一：CPU ↔ SoC

#### 队长负责

- CPU 发出什么请求；
- 请求字段含义；
- CPU 何时认为请求完成；
- CPU 怎样接收 error；
- IRQ 怎样进入 CPU；
- 精确 Trap 行为；
- Cache/MMIO 边界。

#### 队员 A 负责

- SoC 是否按协议接收请求；
- 地址怎样路由；
- Slave 怎样响应；
- 背压；
- 仲裁；
- Response 返回；
- 外设 IRQ 是否正确产生。

#### 契约文件

```text
cpu_soc_interface.md
hxi_interface.md
interrupt_map.md
```

#### 缺陷归属示例

```text
CPU 在 ready=0 时改变地址
→ 队长修 CPU

Crossbar 把 M1 的响应返回给 M0
→ 队员 A 修 Crossbar

协议没有定义写事务何时完成
→ 队长组织决策，双方评审后更新契约
```

### 7.2 边界二：SoC ↔ BSP

#### 队员 A 负责

- 外设地址；
- 寄存器行为；
- 复位值；
- IRQ 条件；
- IRQ 清除；
- Timer 频率；
- 总线访问限制。

#### 队员 B 负责

- 正确使用寄存器；
- 驱动初始化；
- 中断服务；
- 超时处理；
- RT-Thread 设备注册；
- Console 和 Tick。

#### 队长负责

- 批准 Memory Map 和中断方案；
- 解决跨模块架构冲突；
- 确保 CPU 能力满足契约。

#### 缺陷归属示例

```text
驱动写错 UART offset
→ 队员 B 修

RTL STATUS 寄存器行为与文档不一致
→ 队员 A 修 RTL或更正经批准的文档

地址窗口与 RAM 冲突
→ 队长重新做系统决策
```

### 7.3 边界三：BSP ↔ RT-Thread/应用

这部分主要由队员 B 完整拥有。

```text
BSP 提供：
    UART
    Tick
    中断
    heap
    context switch

RT-Thread 提供：
    调度
    线程
    IPC
    软件定时器
    FinSH

应用提供：
    main
    线程演示
    CoreMark 命令
```

如果硬件和 BSP 基础测试已通过，而 FinSH/CoreMark 不能运行，默认由队员 B 继续定位。

---

## 8. 文件所有权矩阵

建议建立 `CODEOWNERS` 或团队内部所有权表。

| 路径/内容 | 主 Owner | Reviewer | 未经批准不得修改者 |
| --- | --- | --- | --- |
| `rtl/core/**` | 队长 | 队员 A | 队员 B |
| CPU 顶层/Wrapper | 队长 | 队员 A | 队员 B |
| `rtl/bus/**` | 队员 A | 队长 | 队员 B |
| `rtl/memory/**` | 队员 A | 队长 | 队员 B |
| `rtl/peripheral/**` | 队员 A | 队长、队员 B | 队员 B 仅评审寄存器 |
| `rtl/clk_rst/**` | 队员 A | 队长 | 队员 B |
| `soc_top/student_top` | 队长 | 队员 A | 队员 B |
| `memory_map.md` | 队长 | A、B | 无人可单方面修改 |
| 外设寄存器文档 | 队员 A | B、队长 | 无人可绕过评审 |
| `software/startup/**` | 队员 B | 队长 | 队员 A |
| `software/bsp/**` | 队员 B | 队长、队员 A | 队员 A 仅评审硬件使用 |
| `software/rt-thread/**` | 队员 B | 队长 | 队员 A |
| `applications/**` | 队员 B | 队长 | 队员 A |
| `coremark/**` | 队员 B | 队长 | 算法文件原则上无人修改 |
| 软件构建/镜像脚本 | 队员 B | 队长 | 队员 A |
| 单元测试 | 对应模块 Owner | 另一人 | 不得只由队长补 |
| 系统回归脚本 | 队员 B | 队长 | A 提供硬件判定条件 |
| Release 记录 | 队长 | A、B | 无人单独发布 |

“Reviewer”不等于接管实现。Reviewer 的任务是检查接口、风险和完成标准。

---

## 9. 使用 RACI/DRI，避免“大家都负责”

### 9.1 DRI 是什么

DRI 可以理解为：

```text
Directly Responsible Individual
直接责任人
```

每一个交付物只能有一个 DRI。

例如：

```text
Machine Timer RTL DRI = 队员 A
RT-Thread Timer Driver DRI = 队员 B
CPU MTIP/MTIE DRI = 队长
```

不能写：

```text
“三个人共同负责 Timer。”
```

因为这通常等于没人对最终闭环负责。

### 9.2 RACI

| 字母 | 含义 |
| --- | --- |
| R | 实际完成工作的人 |
| A | 对结果最终负责的人 |
| C | 需要被咨询的人 |
| I | 需要被告知的人 |

本项目可以简化成：

| 工作项 | 队长 | 队员 A | 队员 B |
| --- | --- | --- | --- |
| 系统架构 | A/R | C | C |
| CPU Core/CSR/Trap | A/R | C | C |
| HXI Crossbar | C | A/R | I |
| BRAM Adapter | C | A/R | I |
| Machine Timer RTL | C | A/R | C |
| UART RTL | C | A/R | C |
| 外设寄存器表 | A/C | R | C |
| `start.S/link.lds` | C | I | A/R |
| RT-Thread Port | C | I | A/R |
| UART/Timer Driver | C | C | A/R |
| CoreMark 接入 | C | I | A/R |
| 单元测试 | C | A/R（SoC） | A/R（软件） |
| 系统回归 | C | C | A/R |
| 顶层集成 | A/R | C | C |
| Release | A/R | C | C |

---

## 10. 交付物必须按“可使用”而不是“写完了”定义

### 10.1 不合格的交付

```text
“Timer 代码写好了，在这个文件里。”
“RT-Thread 我已经复制进来了。”
“UART 仿真看起来差不多。”
“AI 说这个代码应该没问题。”
```

### 10.2 合格的模块交付

每次交付至少包含：

```text
1. 功能说明
2. 接口版本
3. 修改文件
4. 构建/测试命令
5. 测试结果
6. 关键波形或日志
7. 已知限制
8. 对其他模块的要求
9. 出问题时怎样定位
```

### 10.3 推荐交付模板

```text
功能：
    Machine Timer v1

Owner：
    队员 A

输入契约：
    HXI v1.0
    TIMEBASE_FREQ = ...
    TIMER_BASE = ...

输出：
    machine_timer.sv
    timer_regs.md
    timer_tb

已通过：
    counter_increment
    compare_boundary
    irq_level
    rv32_mtimecmp_write
    invalid_offset

未支持：
    MSIP
    多 hart

集成要求：
    CPU irq_timer_i 接 irq_timer_o
    Timer 地址标为 uncached

证据：
    测试日志路径
    波形路径
    Git commit
```

---

## 11. Definition of Done：每块工作什么时候才算完成

### 11.1 RTL 模块 DoD

- [ ] 接口已评审；
- [ ] 复位值明确；
- [ ] 正常路径通过；
- [ ] 等待/背压通过；
- [ ] 非法输入通过；
- [ ] 错误响应通过；
- [ ] 单元测试自动运行；
- [ ] 关键断言存在；
- [ ] 无已知 latch/X 传播；
- [ ] 软件可见行为有文档；
- [ ] 接入顶层后 smoke test 通过；
- [ ] Owner 能解释关键状态机。

### 11.2 BSP/驱动 DoD

- [ ] 地址来自统一头文件；
- [ ] 初始化顺序明确；
- [ ] 轮询和中断路径按要求实现；
- [ ] 超时不会永久死循环；
- [ ] 中断能够正确清除；
- [ ] 裸机测试通过；
- [ ] RT-Thread 中运行通过；
- [ ] 构建无隐藏手工步骤；
- [ ] map/dump 可检查；
- [ ] 日志能判断 PASS/FAIL；
- [ ] Owner 能解释每个 CSR/MMIO 访问。

### 11.3 系统功能 DoD

- [ ] 从干净环境可重复构建；
- [ ] Verilator 可运行；
- [ ] FPGA 可启动；
- [ ] UART 有启动日志；
- [ ] Timer Tick 准确；
- [ ] 线程切换稳定；
- [ ] FinSH 可交互；
- [ ] CoreMark 校验通过；
- [ ] 连续运行达到规定时间；
- [ ] Release 中包含源码、ELF、镜像、日志和版本。

只要 DoD 没完成，任务仍属于原 Owner，不能因为“代码已经发给队长”就视为结束。

---

## 12. Bug 应该归谁：按“最早违反契约的位置”判断

不要按“最后看到错误的位置”分配 Bug。

例如 FinSH 没有输出，最后现象在软件层，但根因可能在：

- UART 地址译码；
- APB Bridge；
- UART TX FIFO；
- CPU MMIO Store；
- BSP Console；
- FinSH 配置。

正确方法是从边界逐级确认：

```text
应用是否调用输出？
    ↓
BSP 是否写 UART 地址？
    ↓
CPU 是否发出正确 HXI Store？
    ↓
Crossbar 是否送到 UART？
    ↓
UART 是否接收并发送？
```

最早违反契约的模块 Owner 负责修复。

### 12.1 示例：UART 没输出

#### 情况 A

反汇编和 Trace 证明软件根本没写 UART：

```text
Owner = 队员 B
```

#### 情况 B

CPU 写地址正确，但在 `ready=0` 时改变 payload：

```text
Owner = 队长
```

#### 情况 C

CPU HXI 请求正确，Crossbar 选错 Slave：

```text
Owner = 队员 A
```

#### 情况 D

所有契约都没说明 UART Base：

```text
这是架构缺口
队长负责组织决策并更新契约
实现仍由对应 Owner 完成
```

### 12.2 示例：RT-Thread 延时不准

逐级检查：

```text
mtime 频率
→ mtimecmp 间隔
→ Timer IRQ 次数
→ CPU 接受次数
→ ISR 调用次数
→ rt_tick 增量
→ RT_TICK_PER_SECOND
```

对应归属：

- `mtime` 频率错误：队员 A；
- CPU 漏接 Timer IRQ：队长；
- ISR 少调用 `rt_tick_increase()`：队员 B；
- 契约中的 `TIMEBASE_FREQ` 不一致：队长组织三方修正。

---

## 13. 跨边界修改必须走变更流程

任何人发现接口不合适时，不能直接修改所有上下游。

### 13.1 变更请求至少写清

```text
当前接口：
    ...

问题：
    ...

证据：
    ...

建议修改：
    ...

影响：
    CPU / SoC / BSP / Test / Linker / Docs

迁移方式：
    ...
```

### 13.2 谁决定

- 局部实现细节：模块 Owner 决定；
- 软件可见寄存器细节：A 提案，B 评审，队长批准；
- CPU/SoC 接口：队长批准，A 评审；
- Memory Map、IRQ、Clock/Reset：队长最终决定；
- RT-Thread 内部配置：B 决定，但不得超出硬件能力。

### 13.3 接口版本

建议给重要契约标版本：

```text
HXI v1.0
Memory Map v1.1
UART Register Map v1.0
Timer Register Map v1.0
CPU IRQ Interface v1.0
```

提交和测试日志注明使用哪个版本。

---

## 14. 队长不接管原则

这是新分工能否生效的关键。

### 14.1 正常情况

当队员遇到自己边界内的困难：

```text
队长提供架构说明和评审
Owner 使用资料、实验和 AI 继续解决
Owner 提交证据和修复
```

队长不能因为：

```text
“我自己做更快”
```

就长期接管。

这样短期看很快，长期会让：

- 队员更不了解系统；
- 所有任务继续等待队长；
- 队长成为单点故障；
- 比赛现场无法并行处理问题。

### 14.2 什么情况下可以临时接管

只有：

- 比赛或交付时间已经进入紧急窗口；
- Owner 连续无法产生有效进展；
- 问题会阻塞整个团队；
- 三人明确同意调整责任；

才可以临时接管。

接管后必须记录：

```text
原 Owner
新 Owner
接管范围
原因
剩余工作
以后由谁维护
```

不能出现队长悄悄修好后，文件名义上仍属于原队员，后续又没人维护。

### 14.3 队长可以做的支持

- 用 15～30 分钟解释接口；
- 帮忙确定最早失败边界；
- 评审 AI 给出的方案；
- 给出一个最小复现框架；
- 指出应阅读的文档；
- 决定架构歧义；

但最终代码、测试和说明仍由 Owner 完成。

---

## 15. 每个队员怎样使用 AI，才不会把负担转给队长

AI 可以提高效率，但前提是每个队员自己是责任人。

### 15.1 每人建立自己的上下文包

队长为所有人提供：

```text
系统方框图
Memory Map
接口文档
目录结构
里程碑
当前测试命令
```

队员再为自己的 AI 补充：

```text
本责任域源文件
模块规格
失败日志
波形
完成标准
禁止修改的边界
```

### 15.2 AI 任务必须限定范围

队员 A 的提示应类似：

```text
只检查 machine_timer 和 HXI Slave。
不得修改 CPU CSR、Memory Map 和 BSP。
需要给出单元测试和寄存器行为说明。
```

队员 B 的提示应类似：

```text
只修改 BSP Timer Driver 和 Trap 分发。
不得修改 RTL 和 CoreMark 算法文件。
需要提供反汇编、日志和构建命令。
```

### 15.3 不能把 AI 输出原样交给队长

Owner 必须做到：

- 能解释修改；
- 知道依赖的接口；
- 自己跑过测试；
- 检查没有跨边界修改；
- 提供结果证据；
- 对后续 Bug 继续负责。

以下交付不接受：

```text
“这是 AI 生成的，我也不太清楚，队长帮忙看一下。”
```

### 15.4 AI 更适合怎样拆任务

```text
读规范
→ 生成模块计划
→ 实现局部功能
→ 自查接口
→ 生成测试点
→ 分析失败日志
→ 修复
```

不要一次要求 AI：

```text
“帮我把整个 SoC 和操作系统做完。”
```

否则它容易跨越责任边界，产生大量队长无法快速审查的修改。

---

## 16. 队员求助时必须提供什么

求助不能只有：

```text
“跑不起来。”
“这个我不会。”
“你帮我看一下。”
```

推荐格式：

```text
目标：
    我正在验证什么

期望：
    正确行为是什么

实际：
    发生了什么

最早失败边界：
    我已经确认到哪一步

证据：
    日志、反汇编、波形、地址和值

已尝试：
    做过哪些检查

我的判断：
    更可能属于哪个模块，为什么

需要的帮助：
    需要接口解释、方案评审，还是跨边界证据
```

这样队长提供的是高价值决策，不是从零代替队员调试。

---

## 17. 主分支和集成规则

### 17.1 主分支必须始终有最低可运行性

每次合入至少满足相应阶段的 Smoke Test：

```text
CPU ISA smoke
SoC RAM smoke
UART hello
Timer IRQ
RT-Thread boot
FinSH
CoreMark
```

项目处于哪个阶段，就要求此前所有 Smoke Test 继续通过。

### 17.2 每个功能在独立分支完成

示例：

```text
feature/hxi-crossbar
feature/machine-timer
feature/mmio-uart
feature/rtthread-port
feature/coremark-command
```

分支合并前：

- Owner 自测；
- Reviewer 检查接口；
- 提供日志；
- 更新文档；
- 标明未完成项。

### 17.3 不允许“大爆炸式集成”

不能三个人各自开发两周，最后一天才合并。

推荐：

```text
小接口先合
模块模型先合
单元测试先合
每 1～2 天形成可运行集成点
```

### 17.4 提交应保持单一责任

不要一个提交同时：

```text
改 CPU
改 Crossbar
改链接脚本
改 CoreMark
改 UART
```

否则失败后无法确认归属，也难以回退单一功能。

---

## 18. 固定的协作节奏

### 18.1 每日十分钟同步

每个人只回答：

```text
昨天完成了什么可验证结果？
今天要完成什么交付物？
当前阻塞在哪个接口？
需要谁提供什么输入？
```

不要把会议变成长时间现场调试。

### 18.2 每两天一次接口检查

检查：

- Memory Map 是否变化；
- HXI 是否变化；
- UART/Timer 寄存器是否变化；
- 软件头文件是否同步；
- 当前主分支通过哪些测试；
- 是否有人跨边界修改。

### 18.3 每周一次里程碑演示

必须演示运行结果，而不是汇报代码行数。

例如：

```text
第 1 周：CPU 通过 Timer IRQ 注入测试
第 2 周：HXI + BRAM + Timer 单元测试通过
第 3 周：裸机 UART + Timer IRQ 上板
第 4 周：RT-Thread 线程与延时
第 5 周：FinSH + CoreMark
```

---

## 19. 分阶段任务表

### 阶段 0：架构冻结

#### 队长

- 系统方框图；
- HXI v1；
- Memory Map v1；
- CPU IRQ v1；
- Clock/Reset v1；
- 里程碑计划。

#### 队员 A

- 评审接口可实现性；
- 给出 SoC 模块清单；
- 给出 BRAM/UART/Timer 资源和风险；
- 建立单元测试框架。

#### 队员 B

- 评审软件可用性；
- 给出 BSP 文件清单；
- 给出链接布局；
- 建立工具链和最小裸机构建。

#### 阶段完成标准

三个人都能讲清五条系统路径，接口文档得到确认。

### 阶段 1：硬件基础

#### 队长

- 稳定 CPU I/D 接口；
- 完成 bus error；
- 完成 CPU Timer IRQ/CSR/Trap；
- 通过 CPU 级中断测试。

#### 队员 A

- HXI Crossbar；
- Default Slave；
- BRAM Adapter；
- Machine Timer；
- 单元测试。

#### 队员 B

- `start.S/link.lds`；
- `hello_uart` 框架；
- `timer_irq_test`；
- ELF/map/dump/镜像工具。

### 阶段 2：裸机系统

#### 队长

- 顶层集成；
- 检查 CPU↔SoC；
- 维护可启动主分支。

#### 队员 A

- MMIO UART；
- APB Bridge；
- GPIO；
- FPGA BRAM/IP；
- 裸机失败时修复 SoC 侧问题。

#### 队员 B

- UART 驱动；
- Timer 驱动；
- Trap 分发；
- 裸机自动日志；
- 裸机失败时修复软件侧问题。

### 阶段 3：RT-Thread

#### 队长

- 解决 CPU 精确中断/上下文边界问题；
- 控制架构不再大改；
- 集成里程碑。

#### 队员 A

- 修复 UART/Timer/IRQ 压力测试暴露的问题；
- 完成中断控制器；
- 检查 CDC 和 FPGA 时序。

#### 队员 B

- RT-Thread RISC-V port；
- Tick；
- Context Switch；
- Heap；
- 线程和软件定时器；
- FinSH。

### 阶段 4：CoreMark 和比赛交付

#### 队长

- 冻结 Release；
- 复核 CoreMark 修改范围；
- 性能/资源权衡；
- 最终答辩和演示流程。

#### 队员 A

- 性能计数/Cache/BRAM 侧验证；
- FPGA 稳定性；
- ILA 预案；
- 资源和时序报告。

#### 队员 B

- 现场 CoreMark `portme`；
- FinSH 命令；
- 编译参数；
- 结果校验；
- Release 软件和操作说明。

---

## 20. 最终比赛现场的分工

### 队长

- 判断现场要求；
- 决定是否调整架构或构建；
- 负责最终版本选择；
- 负责 CPU/集成类故障；
- 对外沟通和演示主线；
- 防止现场临时修改破坏已通过功能。

### 队员 A

- 负责 Vivado、BRAM、时钟、外设和 FPGA 上板；
- 负责 HXI/Timer/UART/IRQ 波形；
- 负责硬件侧性能、资源、时序；
- 准备 ILA 和关键调试信号；
- 出现硬件边界问题时独立处理。

### 队员 B

- 负责现场测评 C 代码接入；
- 负责 CoreMark `portme` 和入口改名；
- 负责 RT-Thread/FinSH 命令；
- 负责工具链、编译、ELF、镜像和日志；
- 负责复核源文件修改范围；
- 出现软件边界问题时独立处理。

这样现场可以真正并行：

```text
队长分析系统和版本风险
队员 A 检查硬件/波形
队员 B 检查软件/构建
```

而不是三个人围着队长等指令。

---

## 21. 防止队长成为单点故障

虽然队长负责总体架构，但至少要有交叉备份：

```text
队员 A：
    能解释 CPU↔SoC 接口、Memory Map 和中断路径

队员 B：
    能解释启动、链接、Trap、Timer 和构建路径

队长：
    能解释两边契约，但不维护全部内部细节
```

每个责任域至少有一位 Reviewer：

```text
CPU：队长 Owner，A Reviewer
SoC RTL：A Owner，队长 Reviewer
软件/BSP：B Owner，队长 Reviewer
外设寄存器：A Owner，B Consumer/Reviewer
系统回归：B Owner，A/队长提供判定信息
```

Reviewer 要能在 Owner 临时不在时完成基本检查，但不要求长期替代 Owner。

---

## 22. 需要建立的团队文档

学习资料可以很详细，但日常工程还需要一套短而明确的“事实文档”。

建议：

```text
docs/
├─ architecture/
│  ├─ system_block_diagram.md
│  ├─ boot_flow.md
│  └─ module_ownership.md
│
├─ interfaces/
│  ├─ hxi_v1.md
│  ├─ cpu_soc_v1.md
│  ├─ interrupt_v1.md
│  └─ clock_reset_v1.md
│
├─ memory_map/
│  ├─ memory_map_v1.md
│  ├─ uart_regs_v1.md
│  ├─ timer_regs_v1.md
│  └─ irq_ctrl_regs_v1.md
│
├─ software/
│  ├─ build.md
│  ├─ linker_layout.md
│  ├─ rtthread_port.md
│  └─ coremark_port.md
│
└─ verification/
   ├─ regression_matrix.md
   ├─ known_issues.md
   └─ releases.md
```

这些文件的 Owner 与模块 Owner 相同。

---

## 23. 三条团队纪律

### 纪律一：没有接口文档，不开始跨模块实现

否则三个人会各自假设地址、时序和中断行为。

### 纪律二：没有测试证据，不交付给下一层

“写完”不算完成，“对方能够按文档使用”才算完成。

### 纪律三：谁拥有模块，谁修复模块

队长负责架构和集成，不是默认救火队员。

跨边界 Bug 先找最早违反契约的位置，再交给对应 Owner。

---

## 24. 最终推荐分工表

| 人员 | 主责任域 | 必须独立闭环 | 不应默认承担 |
| --- | --- | --- | --- |
| 队长 | 架构、CPU、接口、集成、Release | CPU/Trap、契约、顶层、版本 | 重写所有外设和 BSP |
| 队员 A | SoC RTL、总线、存储、外设、FPGA | HXI/BRAM/APB/UART/Timer/IRQ/CDC | 修改 CPU、替 B 写驱动 |
| 队员 B | 启动、BSP、RT-Thread、应用、构建、回归 | Link/Trap/Driver/FinSH/CoreMark/Test | 修改 RTL、改变硬件契约 |

三个人共同负责的不是代码，而是：

```text
理解系统目标
遵守接口
评审里程碑
保护可运行主分支
保证最终演示成功
```

具体交付物必须只有一个 Owner。

---

## 25. 最重要的观念

团队协作效率低，不是因为队长使用 AI 太强，也不一定是因为其他队员完全没有能力。

更常见的原因是：

```text
队长拥有问题的完整上下文
队员只拥有一个模糊任务名称
```

解决方法不是让队长把每一步都拆成指令喂给队员。那仍然会让队长承担全部思考。

真正有效的方式是：

```text
队长提供稳定架构和清晰边界
    ↓
每个队员拥有一块完整责任域
    ↓
Owner 自己学习、使用 AI、实现和验证
    ↓
按接口交付可使用成果
    ↓
集成问题按最早违约点归属
```

队长应当对整个项目负责，但“不亲自完成所有模块”本身就是队长职责的一部分。

只有当队员 A 能独立交付 SoC RTL、队员 B 能独立交付可运行软件，队长才能把精力放在 CPU、系统决策、性能和最终风险上。这样的三人团队才真正比队长一个人使用 AI 更强。

