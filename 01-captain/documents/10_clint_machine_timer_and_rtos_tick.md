# CLINT、Machine Timer 与 RT-Thread Tick：从“计数器”到“操作系统时钟”

> 本文承接 [09：HXI 片上互联、BRAM 存储与 Cache 架构说明](./09_hxi_interconnect_bram_cache_architecture.md)。  
> 面向已经理解 CPU 流水线、CSR、异常等基本概念，但还不熟悉 SoC 外设、中断互联和 RTOS Tick 的读者。  
> 本文以当前 `superScalar` 的 RV32、单核、M-mode、FPGA 环境为背景，但先讲通用原理，再落到当前工程。

---

## 0. 先给出结论

你当前工程中的 `counter.sv` 和 RT-Thread 所需要的 Timer 不是同一种东西。

```text
当前 counter.sv
    = 软件控制开始/停止的毫秒秒表
    = 用来测量一段程序运行了多少毫秒
    = 只能被 CPU 读取
    = 不会主动打断 CPU

RT-Thread 需要的 Machine Timer
    = 持续运行的单调时间基准
    + 一个可由软件设置的比较值
    + 到点后产生机器定时器中断
    = 周期性推动 RT-Thread 的系统 Tick
```

对当前单核 M-mode SoC，第一版不必实现一个面向多核 Linux 的完整 CLINT。建议实现一个**CLINT 兼容或 ACLINT-MTIMER 风格的最小 Machine Timer**：

```text
64 位 mtime       ：持续递增，表示现在是什么时间
64 位 mtimecmp    ：软件写入下一次中断的截止时间
irq_timer         ：当 mtime >= mtimecmp 时拉高
HXI Slave 接口    ：让 CPU 通过 MMIO 读写 mtime/mtimecmp
```

CPU 核还必须配合增加：

```text
irq_timer_i 输入
mie.MTIE
mip.MTIP
异步中断判定
mcause = 0x8000_0007
精确中断入口
mret 返回
```

完整链路是：

```text
Machine Timer RTL
  mtime >= mtimecmp
          │
          ▼
    irq_timer_i / MTIP
          │
          ▼
CPU 检查 mstatus.MIE 与 mie.MTIE
          │
          ▼
在指令边界进入 mtvec
          │
          ▼
RT-Thread Tick ISR
  ├─ 设置下一次 mtimecmp
  ├─ rt_interrupt_enter()
  ├─ rt_tick_increase()
  └─ rt_interrupt_leave()
          │
          ▼
延时、时间片、软件定时器、线程调度向前推进
```

这里最重要的认识是：

> Timer 不负责线程调度。Timer 只负责提供“时间到达”的硬件事件；RT-Thread 收到中断以后，才在软件中更新 Tick、检查延时和决定是否切换线程。

---

## 1. 先把几个容易混淆的“时间”分开

SoC 中经常同时存在很多名字都带 `counter` 或 `timer` 的模块，但它们的目的不同。

| 名称 | 主要回答的问题 | 是否产生中断 | 是否必须一直运行 | 常见位置 |
| --- | --- | ---: | ---: | --- |
| CPU Cycle Counter | CPU 经历了多少周期 | 通常不产生 | CPU 工作时递增 | CPU 核内 CSR |
| Performance Counter | Cache miss、提交、分支错误发生多少次 | 通常不产生 | 视实现而定 | CPU 核内 |
| Stopwatch Counter | 某段程序跑了多少时间 | 通常不产生 | 可手动开始/停止 | SoC 自定义外设 |
| Machine Timer | 现在的单调时间是多少；截止时间是否到达 | **产生 Timer IRQ** | 通常持续运行 | Core-local/SoC |
| General-purpose Timer | 周期中断、单次中断、输入捕获、PWM 等 | 可以产生 | 可配置 | APB 外设 |
| RTC | 现在是几年几月几日几点 | 可产生闹钟中断 | 可能掉电保持 | 独立低速时钟域 |
| Watchdog | 软件是否长时间失去响应 | 超时中断或复位 | 启用后持续运行 | SoC 安全外设 |
| RT-Thread 软件定时器 | 某个软件回调何时执行 | 依赖系统 Tick | 是软件对象 | RT-Thread 内核 |

这些模块不能仅凭“里面都有加法器”就认为是同一个东西。

### 1.1 `mcycle` 是 CPU 周期，不是操作系统时间

`mcycle`/`cycle` 通常每个 CPU 时钟周期加一。因此它很适合：

- 评估 CoreMark；
- 测量一段代码用了多少 CPU 周期；
- 计算 IPC；
- 分析 Cache 和流水线性能。

但它不天然适合作为通用“现实时间”：

- CPU 主频变化后，每个周期代表的秒数会变化；
- CPU 被时钟门控时，cycle 可能停止；
- 多核系统中各核 cycle 可能不完全同步；
- 它本身没有“到某个值时产生中断”的比较器。

当前 `csr_file.sv` 已经能读取 `cycle/cycleh` 和 `instret/instreth`，这对 CoreMark 和性能分析有价值，但它不能替代 RT-Thread Tick Timer。

### 1.2 `mtime` 是单调时间基准

RISC-V Machine Timer 中的 `mtime` 是 64 位、固定频率递增的单调计数值。

“单调”是指：

```text
正常运行时只向前增加，不会因为修改日期、时区而倒退
```

它不直接表示“2026 年 7 月 31 日”，而是类似：

```text
系统复位后已经过去了 12,345,678 个 timebase tick
```

只要软件知道：

```text
TIMEBASE_FREQ = 每秒 mtime 增加多少
```

就可以换算：

```text
秒数 = mtime / TIMEBASE_FREQ
```

### 1.3 `mtimecmp` 不是另一个计数器

`mtimecmp` 是一个**截止时间寄存器**，它自己不递增。

```text
mtime       = 当前时间
mtimecmp    = 下一次需要提醒 CPU 的时间
```

硬件持续做无符号比较：

```text
mtime >= mtimecmp
```

条件成立时，Timer IRQ 置为有效。

例如 `mtime` 每微秒加一：

```text
现在 mtime      = 1,000,000
软件写 mtimecmp = 1,001,000
```

再过 1000 个 timebase tick，也就是 1 ms，机器定时器中断变为 pending。

---

## 2. 什么是 CLINT

CLINT 的常见全称是：

```text
Core-Local Interruptor
```

它最初是 SiFive 平台中广泛使用的一种 core-local 中断设备，通常把两类功能放在同一个 MMIO 地址窗口中：

```text
CLINT
├─ MSIP       ：机器模式软件中断；多核时常用于核间中断 IPI
├─ MTIMECMP   ：每个 hart 一个机器定时器比较值
└─ MTIME      ：整个设备共享的固定频率时间计数器
```

对单核系统，可以直观理解为：

```text
CLINT
├─ 给 CPU 发软件中断（可选）
└─ 给 CPU 发定时器中断（必需）
```

### 2.1 “Core-local”是什么意思

RISC-V 机器模式的主要中断可以先分为：

```text
Core-local 类
├─ Machine Software Interrupt，cause = 3
└─ Machine Timer Interrupt，cause = 7

External 类
└─ Machine External Interrupt，cause = 11
   └─ UART、GPIO、SPI 等通常先进入 PLIC/APLIC
```

Timer 中断通常不通过 PLIC。

```text
Machine Timer ───────────────→ CPU irq_timer_i / MTIP

UART/GPIO/SPI → PLIC/APLIC ──→ CPU irq_external_i / MEIP
```

因此：

- CLINT/Machine Timer 负责本 hart 附近的软件中断和定时器中断；
- PLIC/APLIC 负责多个外设中断源的优先级、屏蔽、仲裁和 claim/complete；
- 两者职责不同，不能把 UART 中断接进 `mtimecmp`，也没必要把 Timer 中断绕进 PLIC。

### 2.2 CLINT、ACLINT 和 MTIMER 的关系

现代资料中还会看到 ACLINT：

```text
ACLINT = Advanced Core Local Interruptor
```

ACLINT 将原来 CLINT 的统一设备拆成更模块化的几个 MMIO 设备：

```text
ACLINT
├─ MTIMER：机器模式固定频率计数与定时器事件
├─ MSWI  ：机器模式软件中断/IPI
└─ SSWI  ：Supervisor 模式软件中断/IPI
```

可以把传统 SiFive CLINT 理解成：

```text
一个 MSWI + 一个 MTIMER
```

对你当前的目标——单核、M-mode、RT-Thread——真正必须的是：

```text
MTIMER
```

`MSIP/MSWI` 对第一版不是必须的，因为当前没有多核 IPI 需求。你可以：

1. 只实现 MTIMER，但模块名明确叫 `machine_timer` 或 `aclint_mtimer`；
2. 在地址空间中预留 `MSIP`，以后扩展；
3. 如果为了兼容常见软件地址布局，实现一个单核最小 `clint`，其中暂时只让 Timer 部分工作。

不要为了名字好听，直接声称实现了“完整标准 CLINT”，却没有说明寄存器映射、hart 数量和软件中断功能。

### 2.3 RISC-V 是否规定 CLINT 的固定地址

没有。RISC-V 特权架构规定了 Machine Timer 对软件应呈现的核心语义，但具体 MMIO 基地址属于平台规范。

很多 SiFive/QEMU 风格平台采用常见的 CLINT 布局：

| 相对 CLINT 基地址的偏移 | 单核用途 |
| ---: | --- |
| `0x0000` | `msip[0]` |
| `0x4000` | `mtimecmp[0]` 低 32 位 |
| `0x4004` | `mtimecmp[0]` 高 32 位 |
| `0xBFF8` | `mtime` 低 32 位 |
| `0xBFFC` | `mtime` 高 32 位 |

但这是一种常见平台兼容布局，不是 ISA 强制要求的唯一地址。

对当前项目，建议优先选择以下两种策略之一：

#### 策略 A：采用常见 CLINT 兼容布局

优点：

- 容易借鉴 QEMU、SiFive 和其他开源 BSP；
- 软件宏和调试经验比较丰富；
- 以后加入 `msip` 比较自然。

代价：

- 会占用一个比较大的稀疏地址窗口；
- 对单核小 SoC 来说，很多地址没有实际寄存器。

#### 策略 B：采用项目自定义的紧凑 MTIMER 窗口

例如只定义：

```text
+0x00  MTIME_LO
+0x04  MTIME_HI
+0x08  MTIMECMP_LO
+0x0C  MTIMECMP_HI
```

优点是简单；代价是所有 BSP 宏和测试都要跟着你的平台定义，不能直接声称兼容 CLINT 地址布局。

**本文建议**：如果地址空间足够，采用 CLINT 兼容偏移；基地址仍由你自己的 `memory_map_pkg.sv` 冻结。这样有利于复用软件，但文档中必须写清楚“兼容了哪些寄存器”，而不是只写一个 CLINT 名称。

---

## 3. Machine Timer 到底负责什么

### 3.1 它负责“在未来某个单调时间点提醒 CPU”

Machine Timer 的硬件职责可以压缩成四件事：

1. 按固定频率推进 `mtime`；
2. 接受 CPU 对 `mtime` 和 `mtimecmp` 的 MMIO 读写；
3. 比较 `mtime` 与 `mtimecmp`；
4. 把比较结果作为机器定时器中断 pending 信号送给 CPU。

它不负责：

- 保存线程寄存器；
- 选择下一个线程；
- 维护 RT-Thread 线程链表；
- 实现 `rt_thread_delay()`；
- 执行软件定时器回调；
- 统计 CoreMark 算法结果；
- 产生日历日期；
- 仲裁 UART/GPIO 等外部中断。

### 3.2 Timer IRQ 应该是电平，不是一个单周期脉冲

RISC-V Machine Timer 的语义是：

```text
MTIP = (mtime >= mtimecmp)
```

因此它是 pending 电平：

```text
mtime < mtimecmp  → irq_timer = 0
mtime >= mtimecmp → irq_timer = 1
```

进入中断后，软件通常把 `mtimecmp` 写到未来：

```text
mtimecmp = 下一次截止时间
```

这会让：

```text
mtime < 新的 mtimecmp
```

于是中断撤销。

如果只输出一个周期的脉冲，会出现：

- CPU 当时关闭中断，脉冲被错过；
- CPU 正在处理更高优先级 Trap，脉冲被错过；
- CDC 同步器没有采到短脉冲；
- 软件还没来得及处理，硬件事件已经消失。

所以 `irq_timer` 应当保持有效，直到软件把比较值推到未来。这也是“pending”而不是“pulse”的含义。

### 3.3 总线接口和中断线是两条不同的路径

这是理解 SoC Timer 最关键的一点。

```text
路径 1：配置/读取路径

CPU Load/Store
    ↓
D-cache MMIO bypass
    ↓
HXI-D Master
    ↓
HXI Crossbar
    ↓
Machine Timer HXI Slave
    ↓
读 mtime / 写 mtimecmp


路径 2：事件通知路径

mtime >= mtimecmp
    ↓
irq_timer_o
    ↓
CPU irq_timer_i
    ↓
mip.MTIP
```

CPU 通过总线告诉 Timer“下一次什么时候叫我”；Timer 到点以后通过专用中断线通知 CPU。

不能只实现 MMIO 寄存器而不接中断线；那样 CPU 只能不断轮询：

```c
while (mtime < deadline)
{
    /* 一直占用 CPU */
}
```

这不适合作为 RTOS Tick。

---

## 4. Timer 如何推动 RT-Thread

假设：

```text
TIMEBASE_FREQ      = 1,000,000 Hz
RT_TICK_PER_SECOND = 1,000
```

那么每个 RT-Thread Tick 对应：

```text
TICKS_PER_OS_TICK
    = TIMEBASE_FREQ / RT_TICK_PER_SECOND
    = 1,000
```

也就是每 1 ms 触发一次机器定时器中断。

### 4.1 启动阶段

BSP 在 `rt_hw_tick_init()` 或等价函数中完成：

```text
1. 读取当前 mtime
2. next_deadline = mtime + 1000
3. 写入 mtimecmp
4. 使能 mie.MTIE
5. 确保 mtvec、Trap 栈和入口已经准备好
6. 在合适时机使能 mstatus.MIE
```

### 4.2 每次 Tick 中断

当 `mtime >= mtimecmp`：

```text
Machine Timer 拉高 MTIP
    ↓
CPU 在精确指令边界接受中断
    ↓
mcause = 0x8000_0007
    ↓
PC 跳到 mtvec
    ↓
Trap 入口保存上下文
    ↓
Timer ISR
```

ISR 中通常先安排下一次截止时间，再通知 RT-Thread：

```c
next_deadline += ticks_per_os_tick;
write_mtimecmp(next_deadline);

rt_interrupt_enter();
rt_tick_increase();
rt_interrupt_leave();
```

`rt_tick_increase()` 会推动：

- RT-Thread 全局 Tick；
- 当前线程时间片；
- `rt_thread_delay()` 等待时间；
- 软件定时器检查；
- 必要时的调度请求。

真正的线程上下文切换可能发生在中断退出路径，而不是由 Timer RTL 完成。

### 4.3 Tick 中断不等于“每 1 ms 必须切一次线程”

每次 Tick 都进入内核，但是否切线程取决于：

- 当前线程时间片是否用完；
- 是否有更高优先级线程变为 Ready；
- 某个延时线程是否到期；
- 调度器当前状态。

所以实际过程是：

```text
Timer 每 1 ms 提醒一次
    ↓
RT-Thread 检查软件状态
    ↓
可能继续执行原线程，也可能切换线程
```

### 4.4 RT-Thread 软件定时器不是额外的硬件 Timer

如果应用创建十个 RT-Thread 软件定时器，不代表 SoC 必须实例化十个硬件计时器。

```text
一个 Machine Timer 周期中断
    ↓
一个全局 rt_tick
    ↓
RT-Thread 管理许多软件定时器
```

软件定时器只是内核中的对象和有序队列。硬件提供基础节拍，软件在其上复用出大量逻辑定时器。

---

## 5. 当前 `counter.sv` 到底是什么

当前模块的核心行为是：

```text
cnt_enable_cpu = 1
    ↓
cnt_clk 域开始计数
    ↓
每累计 50,000 个 cnt_clk 周期
    ↓
cnt_ms_bin 加 1
    ↓
通过 Gray Code 跨时钟域到 cpu_clk
    ↓
CPU 从 0x8020_0050 读取毫秒值
```

软件写入：

```text
0x8000_0000 → 开始计数
0xFFFF_FFFF → 停止计数
```

因此它是一个测量区间的秒表。

还要注意：当前 STOP 只会停止计数并清零不足 1 ms 的分频计数，`cnt_ms_bin` 本身不会被清零；下一次 START 会从上次累计值继续。也就是说，它目前更准确地说是一个“可暂停/继续的累计毫秒计数器”。如果比赛程序希望每次 START 都从 0 计时，还需要额外的清零命令或由软件记录起止读数并相减。

### 5.1 它现在适合做什么

- 测量一段测试程序运行了多少毫秒；
- 在数码管/调试界面显示耗时；
- 作为原有比赛环境的性能测量外设；
- 检验简单 CDC 和 MMIO 读取。

### 5.2 它为什么不能直接当 RT-Thread Timer

| RTOS Timer 需求 | 当前 `counter.sv` |
| --- | --- |
| 上电后持续提供单调时间 | 只有收到 START 才运行 |
| 通常为 64 位 | 只有 32 位毫秒计数 |
| 可设置下一次截止时间 | 没有 `mtimecmp` |
| 到点产生 Timer IRQ | 没有 IRQ 输出 |
| IRQ 保持到软件重新编程 | 不存在 pending 逻辑 |
| CPU 有 MTIP/MTIE 接口 | 当前 CPU 没有 Timer IRQ 输入 |
| 软件可周期性安排下次 Tick | 只能 START/STOP |

此外，当前代码只有在 `cnt_clk = 50 MHz` 时：

```text
49,999 + 1 = 50,000 个周期 = 1 ms
```

如果 `cnt_clk` 不是 50 MHz，读数就不是毫秒。这个频率假设应当变成明确参数或 SoC 规范，而不能隐藏在常数 `49999` 中。

### 5.3 32 位毫秒计数为什么仍然不理想

32 位毫秒计数约在 49.7 天后回绕。对比赛程序可能够用，但 RISC-V Machine Timer 要求的核心软件接口是 64 位精度。

64 位、1 MHz 的 `mtime` 大约需要数十万年才回绕，软件几乎不用处理短期溢出。

### 5.4 当前 Gray Code CDC 有什么意义

`counter.sv` 中：

```text
二进制计数 → Gray Code → 两级同步 → Gray 转二进制
```

目的是让一个多位、持续变化的计数值安全跨越 `cnt_clk` 和 `cpu_clk`。

这对“读一个自由运行计数值”是合理思路之一，但把 Machine Timer 放到独立时钟域后，问题会更复杂：

- CPU 写 `mtimecmp` 要跨到 Timer 域；
- CPU 读 `mtime/mtimecmp` 要获得一致快照；
- `irq_timer` 要从 Timer 域同步到 CPU 域；
- 64 位数据跨域不能简单对每一位各放两个触发器；
- HXI 请求/响应还处于 CPU 时钟域。

因此第一版建议把 Machine Timer、HXI Slave 和 CPU 放在同一 `cpu_clk` 域，用**时钟使能脉冲**控制 `mtime` 的递增频率，而不是立刻引入独立 Timer 时钟域。

---

## 6. 常见 SoC 中 Machine Timer 的 RTL 结构

一个最小的单核 Machine Timer 可以分成五部分：

```text
machine_timer
├─ timebase_divider      固定频率时基/时钟使能
├─ mtime_reg             64 位单调计数器
├─ mtimecmp_reg          64 位截止时间
├─ compare_and_irq       比较并输出电平中断
└─ hxi_slave_regs        MMIO 寄存器访问
```

### 6.1 固定频率时基

假设 CPU/SoC 时钟固定为 50 MHz，可以有两种做法。

#### 做法 A：`mtime` 每个 CPU 周期加一

```text
TIMEBASE_FREQ = 50 MHz
mtime         = mtime + 1，每拍一次
```

RT-Thread 1 kHz Tick 的比较间隔是：

```text
50,000,000 / 1,000 = 50,000
```

优点：

- RTL 最简单；
- 计时分辨率高；
- 没有额外分频器。

缺点：

- 修改 CPU/SoC 主频时必须同步修改 BSP 的 `TIMEBASE_FREQ`；
- 如果未来 CPU 动态变频，`mtime` 就不再代表固定现实时间。

#### 做法 B：用 clock-enable 产生 1 MHz 时基

```text
cpu_clk = 50 MHz
每 50 个 cpu_clk 周期产生一个 timebase_ce
mtime 只在 timebase_ce = 1 时加一
TIMEBASE_FREQ = 1 MHz
```

RT-Thread 1 kHz Tick 的比较间隔固定为：

```text
1,000,000 / 1,000 = 1,000
```

这里的 `timebase_ce` 是一个单周期使能信号，不是用普通 RTL 随意生成的新时钟：

```text
推荐：always_ff @(posedge cpu_clk) if (timebase_ce) mtime <= mtime + 1;

不推荐：用组合逻辑翻转出 timer_clk，再把它当新时钟到处使用
```

对当前 FPGA 项目，**第一版推荐做法 B**。它容易理解，也能让软件中的时间单位接近微秒。

但如果比赛阶段会频繁把 `cpu_clk` 从 50 MHz 改到 100/150/200 MHz，仍要同步修改分频参数。更成熟的版本可以从板上稳定参考时钟产生 timebase，再解决 CDC；那是第二阶段工作。

### 6.2 64 位 `mtime`

概念行为是：

```text
复位时 mtime = 0

每次 timebase_ce：
    mtime = mtime + 1
```

RISC-V 规范将 `mtime` 描述为固定频率、64 位、可由机器模式软件访问的内存映射寄存器。

是否允许软件写 `mtime` 必须写进你的 SoC 规范。为了接近规范和常见 CLINT，可以实现读写；但 RT-Thread 正常运行一般只读 `mtime`，不应频繁修改它。

### 6.3 64 位 `mtimecmp`

`mtimecmp` 由软件写入：

```text
mtimecmp = 下一次截止时间
```

对单核只有一个：

```text
mtimecmp[0]
```

多核时通常每个 hart 有一个独立的 `mtimecmp`，但可以共享同一个 `mtime`：

```text
共享 mtime
├─ mtimecmp[0] → hart 0 timer IRQ
├─ mtimecmp[1] → hart 1 timer IRQ
└─ ...
```

### 6.4 64 位比较器和中断输出

概念等式是：

```text
irq_timer_o = (mtime >= mtimecmp)
```

应使用无符号比较。

在当前 50 MHz FPGA 目标上，64 位比较通常不构成最困难的路径。若以后频率很高，可以把比较结果寄存，但要保持：

- 到点后最终会反映到 MTIP；
- 中断是电平；
- 写入未来比较值后最终会撤销；
- 软件和验证能够接受规定的有限传播延迟。

### 6.5 复位值

建议项目第一版采用确定性复位：

```text
mtime    = 0
mtimecmp = 0xFFFF_FFFF_FFFF_FFFF
irq      = 0
```

这样复位释放后不会立刻发生 Timer IRQ。

这是当前项目的工程建议。若声称严格兼容某一份外部 CLINT/ACLINT 规范，还要逐项核对其复位要求，并在平台文档中说明差异。

---

## 7. RV32 访问 64 位 Timer 寄存器的特殊问题

你的数据总线一次只有 32 位，而 `mtime` 和 `mtimecmp` 都是 64 位，所以一次软件访问只能读写一半。

### 7.1 安全读取 `mtime`

错误做法：

```text
先读低 32 位，再读高 32 位
```

因为两次读取之间，低 32 位可能从：

```text
0xFFFF_FFFF → 0x0000_0000
```

同时高 32 位加一，最后拼出的值可能前后不一致。

常见安全读取方法：

```text
high_1 = MTIME_HI
low    = MTIME_LO
high_2 = MTIME_HI

如果 high_1 != high_2：
    重新读取
否则：
    time = (high_1 << 32) | low
```

### 7.2 安全写入 `mtimecmp`

如果希望写入：

```text
new_cmp = {new_hi, new_lo}
```

不能随意先写低、再写高。中间状态可能暂时小于当前 `mtime`，从而产生意外中断。

RV32 常见安全顺序是：

```text
1. MTIMECMP_LO = 0xFFFF_FFFF
2. MTIMECMP_HI = new_hi
3. MTIMECMP_LO = new_lo
```

第一步先把比较值的低半部抬到最大，避免在更新过程中形成一个过早的截止时间。

Timer RTL 必须支持对 64 位寄存器两个 32 位半部的独立访问，并正确处理 HXI 的 `wstrb`。

第一版可以规定：

```text
只支持 32 位对齐访问
写寄存器要求 wstrb = 4'b1111
其他访问返回 bus error
```

也可以完整支持 byte strobe，但验证工作会明显增加。无论采用哪种方式，都要把规则写进寄存器规范和 BSP。

---

## 8. Timer 应该怎样接入 09 号文档中的 HXI SoC

推荐的数据通路如下：

```text
CPU Pipeline
├─ IFU / I-side
└─ LSU / D-cache
       │
       ├─ 普通 RAM 地址 → D-cache
       │
       └─ Timer MMIO 地址 → 强制 bypass Cache
                              │
                              ▼
                         HXI-D Master
                              │
                              ▼
                         HXI Crossbar
                              │
                              ▼
                     hxi_machine_timer_slave
                              │
                              ├─ mtime
                              └─ mtimecmp
```

中断路径则是：

```text
hxi_machine_timer_slave.irq_timer_o
                    │
                    ▼
cpu_subsystem.irq_timer_i
                    │
                    ▼
csr_file 中的 mip.MTIP
```

### 8.1 直接挂 HXI 还是放到 APB

两种做法都能工作。

#### 直接作为 HXI Slave

```text
HXI Crossbar
└─ Machine Timer HXI Slave
```

优点：

- 没有 HXI-to-APB 的额外状态机延迟；
- Timer 与 core-local 中断模块层次清楚；
- 容易与 09 号方案中的独立 CLINT/IRQ Controller 窗口一致；
- 64 位拆分访问和错误响应可由专用适配器明确处理。

#### 作为 APB 外设

```text
HXI Crossbar
└─ HXI-to-APB
   └─ APB Machine Timer
```

优点：

- 复用统一的低速寄存器外设框架；
- Timer 的访问频率其实很低，性能通常不是问题；
- UART/GPIO/Timer 可以使用类似的寄存器模板。

需要注意：

> Timer IRQ 无论寄存器挂在 HXI 还是 APB，都应该通过专用中断线送入 CPU。总线层次不会改变中断的 cause。

结合当前 09 号架构，建议第一版将 Machine Timer 作为独立 HXI Slave。不是因为它的带宽很高，而是为了让 core-local Timer 的地址、64 位寄存器和 IRQ 语义更集中、更容易验证。

### 8.2 Timer 地址必须是 non-cacheable MMIO

Timer 地址区必须：

```text
non-cacheable
non-executable
strongly ordered / device-like
禁止预取
禁止合并
```

原因包括：

- 连续读取 `mtime` 必须真的访问硬件；
- 写 `mtimecmp` 必须按软件指定顺序到达；
- StoreBuffer 不能把三次 RV32 写更新乱序；
- D-cache 不能返回旧的 `mtime`；
- 不能把中断清除写延后到不可预测的时间。

对 CPU/D-cache/StoreBuffer 的要求是：

```text
识别 Timer MMIO 地址
    ↓
绕过 D-cache
    ↓
等待更早的设备写按规定完成
    ↓
按程序顺序发出本次 MMIO
```

### 8.3 建议的 Timer 模块接口

以下是职责级接口，不是要求你立即照抄的完整 RTL：

```text
输入：
    clk
    rst

    hxi_req_valid
    hxi_req_addr
    hxi_req_write
    hxi_req_wdata[31:0]
    hxi_req_wstrb[3:0]
    hxi_rsp_ready

输出：
    hxi_req_ready
    hxi_rsp_valid
    hxi_rsp_rdata[31:0]
    hxi_rsp_err

    irq_timer_o
```

如果 HXI 第一版限制每个 Master 只有一个 outstanding，Timer Slave 也可以采用简单状态机：

```text
IDLE
  接收请求并锁存地址/方向/数据
    ↓
ACCESS
  完成寄存器读写
    ↓
RESPONSE
  保持 rsp_valid，直到 rsp_ready
```

如果寄存器访问可以单周期接受，也仍要遵守：

```text
rsp_valid && !rsp_ready 时，rsp_rdata/rsp_err 保持不变
```

---

## 9. CPU 核需要为 Timer 中断实现什么

Timer 不是只改 SoC 外设就能使用。当前 CPU 还没有异步中断入口，必须补齐核内逻辑。

### 9.1 当前工程的具体缺口

根据当前源码：

- `myCPU.sv`/`core_top.sv` 没有 `irq_timer_i` 输入；
- `csr_file.sv` 有 `mstatus/mtvec/mscratch/mepc/mcause/mtval`；
- 没有 `mie`；
- 没有 `mip`；
- `mcause` 当前由 5 位同步异常 cause 拼成，不能表达 bit 31 的“这是中断”；
- 当前 Trap 由提交项中的同步异常触发；
- 没有在指令边界注入异步 Timer Interrupt 的路径。

所以新增 `machine_timer.sv` 后，如果不改 CPU，Timer 即使拉高 IRQ，CPU 也完全看不见。

### 9.2 CPU 顶层端口

至少增加：

```text
input irq_timer_i
```

为了完整规划，可以同时预留：

```text
input irq_software_i
input irq_timer_i
input irq_external_i
```

映射关系：

| CPU 输入 | `mip` 位 | `mie` 位 | `mcause` |
| --- | --- | --- | --- |
| `irq_software_i` | `MSIP` bit 3 | `MSIE` bit 3 | `0x8000_0003` |
| `irq_timer_i` | `MTIP` bit 7 | `MTIE` bit 7 | `0x8000_0007` |
| `irq_external_i` | `MEIP` bit 11 | `MEIE` bit 11 | `0x8000_000B` |

第一版只实现 Timer，也应让未实现的两路固定为 0，接口语义保持明确。

### 9.3 `mip.MTIP` 应反映 Timer 电平

概念关系是：

```text
mip.MTIP = irq_timer_i
```

对这种外部硬件驱动的 pending 位，软件通常不能通过写 `mip` 清除它。真正的清除方式是：

```text
写 mtimecmp 到未来
```

如果 CSR 软件把 `mip.MTIP` 写成 0，但 Timer 仍满足 `mtime >= mtimecmp`，它应继续 pending。

### 9.4 中断使能条件

在当前单核、仅 M-mode 目标中，可以先理解为：

```text
timer_pending = mip.MTIP
timer_enable  = mie.MTIE
global_enable = mstatus.MIE

take_timer_interrupt
    = timer_pending
    && timer_enable
    && global_enable
    && 当前到达可精确接收中断的指令边界
```

仅仅 `irq_timer_i = 1` 不等于 CPU 必须立刻跳走。CPU 要完成当前可提交状态，并遵循全局和局部中断使能。

### 9.5 精确异步中断

同步异常与某一条指令绑定：

```text
非法指令、地址未对齐、ECALL……
```

Timer Interrupt 是异步事件，不属于某一条正在执行的指令。CPU 应在一个精确的指令边界接受它：

```text
所有更老指令已经完成/提交
所有更年轻指令不会产生可见副作用
mepc 指向返回后应继续执行的下一条指令
```

对当前提交式结构，可以把中断看成在安全提交边界生成的一次“合成 Trap”，但不能把它错误地塞进某一条普通指令的同步异常字段。

同时发生同步异常和 Timer IRQ 时，第一版建议：

```text
同步异常优先
Timer IRQ 保持 pending
异常处理返回并重新允许中断后，再接受 Timer IRQ
```

因为 Timer IRQ 是电平，不会因为推迟几个周期而丢失。

### 9.6 进入 Timer Trap 时 CSR 的变化

典型 M-mode 行为：

```text
mepc          = 返回地址
mcause[31]    = 1，表示 Interrupt
mcause[30:0]  = 7，表示 Machine Timer Interrupt
mtval         = 0
mstatus.MPIE  = 进入前的 mstatus.MIE
mstatus.MIE   = 0
mstatus.MPP   = 进入前特权级（当前为 M）
PC            = mtvec BASE（先只实现 Direct 模式）
```

`mret` 时恢复：

```text
mstatus.MIE  = mstatus.MPIE
mstatus.MPIE = 1
PC           = mepc
```

Timer IRQ 的接入必须与现有同步异常、`mret`、流水线 flush 和 StoreBuffer 精确副作用统一验证。

---

## 10. 一个适合当前项目的推荐方案

### 10.1 第一版范围

```text
单核
RV32
仅 M-mode
mtime 64 位
mtimecmp 64 位
1 MHz timebase
1 路机器定时器中断
HXI Slave
无 MSIP
无 S-mode Timer
无多核 IPI
无动态变频
```

这已经足以：

- 裸机定时器中断；
- RT-Thread 1 kHz Tick；
- `rt_thread_delay()`；
- 时间片轮转；
- RT-Thread 软件定时器；
- 在 FinSH 中运行 CoreMark 时保持系统时钟推进。

### 10.2 推荐层次

```text
rtl/
├─ top/
│  ├─ fpga_top.sv
│  └─ soc_top.sv
│
├─ cpu/
│  ├─ cpu_subsystem.sv
│  └─ csr_trap/
│     ├─ csr_file.sv
│     ├─ irq_pending_ctrl.sv
│     └─ trap_ctrl.sv
│
├─ bus/
│  ├─ hxi_pkg.sv
│  ├─ hxi_crossbar.sv
│  └─ hxi_default_slave.sv
│
├─ peripheral/
│  ├─ machine_timer.sv
│  ├─ timer_reg_if.sv        # 可选；项目小也可合入 machine_timer
│  ├─ apb_uart.sv
│  ├─ apb_gpio.sv
│  └─ interrupt_controller.sv
│
└─ memory/
   └─ memory_map_pkg.sv
```

软件侧：

```text
bsp/
├─ board.c
├─ drv_timer.c
├─ drv_uart.c
├─ trap_gcc.S
├─ interrupt.c
├─ rtconfig.h
└─ link.lds
```

### 10.3 地址映射

09 号文档只为 CLINT/Machine Timer 预留了独立窗口，没有冻结基地址。现在应在 `memory_map_pkg.sv` 和 BSP 头文件中同时定义唯一来源。

如果采用 CLINT 兼容布局，规范中应明确写：

```text
CLINT_BASE              = 待项目冻结
MSIP0                   = CLINT_BASE + 0x0000（可预留）
MTIMECMP0_LO            = CLINT_BASE + 0x4000
MTIMECMP0_HI            = CLINT_BASE + 0x4004
MTIME_LO                = CLINT_BASE + 0xBFF8
MTIME_HI                = CLINT_BASE + 0xBFFC
```

如果采用紧凑自定义布局，就明确标注：

```text
不是 SiFive CLINT 地址兼容
BSP 只能使用本项目头文件中的地址
```

不要让以下三处各写一份不同的 magic number：

```text
HXI 地址译码
Timer RTL
BSP C 头文件
```

建议从一份设计规范生成或人工同步审查：

```text
memory_map_pkg.sv
soc_memory_map.h
link.lds（如果相关）
仿真模型/测试程序
```

### 10.4 Timer 频率参数

硬件规范中必须冻结：

```text
SOC_CLK_FREQ
TIMEBASE_FREQ
RT_TICK_PER_SECOND
```

推荐示例：

```text
SOC_CLK_FREQ       = 50,000,000 Hz
TIMEBASE_FREQ      = 1,000,000 Hz
RT_TICK_PER_SECOND = 1,000 Hz
TIMEBASE_DIV       = 50
MTIME_DELTA        = 1,000
```

其中前两个属于 SoC/BSP 硬件契约，第三个属于 RT-Thread 配置。必须满足：

```text
TIMEBASE_FREQ % RT_TICK_PER_SECOND == 0
```

否则简单整数分频会造成系统 Tick 误差；更复杂的实现可以用相位累加或交替间隔补偿，但第一版没必要。

---

## 11. Machine Timer 和通用 Timer 有什么区别

在 MCU/SoC 资料中，“Timer”有时指专门的 RISC-V Machine Timer，有时指 APB 上的通用定时器。

### 11.1 Machine Timer

核心模型：

```text
自由运行向上计数器 + 绝对时间比较寄存器
```

主要用途：

- OS Tick；
- 固件超时；
- 调度下一个定时事件；
- 为 M-mode 提供标准 Timer Interrupt。

### 11.2 通用定时器 GPT

一个 APB General-Purpose Timer 可能包含：

```text
CTRL
PRESCALER
COUNTER
RELOAD
COMPARE
INT_STATUS
INT_ENABLE
```

支持：

- 单次倒计时；
- 周期自动重装；
- 多个 compare channel；
- 输入捕获；
- 输出比较；
- PWM；
- 边沿计数。

它也能用来产生 RTOS Tick，但软件接口不是 RISC-V `mtime/mtimecmp` 模型，需要 BSP 驱动自行适配。

### 11.3 当前项目为什么优先 Machine Timer

- 与 RISC-V M-mode Timer Interrupt 语义直接匹配；
- RT-Thread Tick 只需要单调计数与截止时间；
- 不需要 PWM、捕获等额外功能；
- 容易和其他 RISC-V BSP、QEMU 测试对照；
- 后续若引入 SBI/S-mode，也更容易理解系统分层。

如果比赛还要求 PWM、输入捕获或多个外设定时通道，再额外设计 APB GPT，不要把所有功能强塞进 CLINT。

---

## 12. Timer、RTC、Watchdog 和 Counter 的常见关系

### 12.1 Timer 不是 RTC

Machine Timer 通常从复位后的 0 开始：

```text
mtime = 8,000,000
```

只能说明经过了多少 timebase tick，不能直接说明日期。

RTC 可能有：

```text
秒、分、时、日、月、年
闹钟
掉电保持
32.768 kHz 晶振
```

RT-Thread 的系统 Tick 能够支持相对延时；如果应用需要真实日期，还要有 RTC 或由外部同步时间。

### 12.2 Timer 不是 Watchdog

Watchdog 的逻辑是：

```text
软件必须周期性“喂狗”
如果长时间没喂
    → 中断或复位整个系统
```

它用于发现软件卡死，不用于正常线程时间片调度。

### 12.3 CoreMark 计时优先用什么

CoreMark 希望测量运行时间或周期数：

- 如果需要高分辨率性能数据，可使用 `cycle/cycleh`；
- 如果规则要求墙上时间，可使用 `mtime` 和已知 `TIMEBASE_FREQ`；
- 当前 START/STOP `counter.sv` 也可作为比赛指定秒表，但精度只有毫秒；
- 不建议用 RT-Thread 的 1 ms Tick 作为高精度 CoreMark 计时源。

所以最终 SoC 中保留多个计数器是合理的：

```text
CPU cycle/perf counters   → 性能分析
Machine Timer             → RTOS Tick 与单调时间
原有 stopwatch counter   → 若比赛接口仍要求，可保留
```

它们不是重复实现，而是测量对象不同。

---

## 13. 软件接入时不要直接照抄 RT-Thread 的 RV64/SBI 代码

RT-Thread 官方仓库中某些 RISC-V BSP 使用：

```text
rdtime
sbi_set_timer()
Supervisor Timer Interrupt
```

那种架构通常是：

```text
RT-Thread 运行在 S-mode
    ↓ SBI 调用
M-mode 固件设置真实 mtimecmp
```

而当前项目计划是：

```text
RT-Thread 直接运行在 M-mode
没有 OpenSBI
没有 S-mode
```

因此当前 BSP 更直接：

```text
M-mode 软件直接读 MMIO mtime
M-mode 软件直接写 MMIO mtimecmp
使能 mie.MTIE
处理 mcause = 0x8000_0007
```

可以参考官方代码的“计算下一次 Tick、调用 `rt_tick_increase()`”思路，但不能直接复制 `sbi_set_timer()`、`sie.STIE` 和 RV64 的单条 64 位访问。

当前 RV32 必须特别处理：

- 32 位总线拆分 64 位寄存器；
- `mtime` 稳定读取；
- `mtimecmp` 安全更新；
- M-mode 的 `mie.MTIE`；
- Machine Timer cause 7。

---

## 14. RTL 验证应该怎样做

不要直接以“RT-Thread 能启动”作为 Timer 的第一个测试。应该逐层验证。

### 14.1 Timer 单元级测试

#### 计数

```text
复位后 mtime = 0
每 TIMEBASE_DIV 个 clk，mtime 精确加 1
其他周期保持
```

#### 比较边界

```text
mtime = mtimecmp - 1 → irq = 0
mtime = mtimecmp     → irq = 1
mtime > mtimecmp     → irq = 1
```

#### 中断撤销

```text
irq = 1 时，把 mtimecmp 写到未来
经过规定传播延迟后 irq = 0
```

#### 电平特性

```text
CPU 不响应时，irq 必须一直保持
不能只出现一个周期
```

#### 复位

```text
复位期间 irq = 0
释放复位后不能立刻误中断
```

#### 回绕

把 `mtime` 初始化到：

```text
0xFFFF_FFFF_FFFF_FFFD
```

检查递增和无符号比较的边界行为。实际运行几乎不会等到回绕，但 RTL 运算仍要定义清楚。

### 14.2 HXI/MMIO 测试

至少覆盖：

```text
读 MTIME_LO
读 MTIME_HI
写 MTIMECMP_LO
写 MTIMECMP_HI
非法偏移返回 error
非对齐访问返回 error
不支持的 wstrb 返回 error
rsp_ready 拉低时响应保持
连续访问不丢失
未映射地址不会被 Timer 错误接收
```

还要按 RV32 安全序列实际写一次 `mtimecmp`，确认中间不会产生伪中断。

### 14.3 CPU 中断测试

分别验证：

```text
MTIP=1，MTIE=0             → 不进入中断
MTIP=1，MTIE=1，MIE=0      → 不进入中断
MTIP=1，MTIE=1，MIE=1      → 在精确边界进入
```

进入后检查：

```text
mcause == 0x8000_0007
mepc   == 正确的返回 PC
mstatus.MPIE/MIE 正确
PC     == mtvec BASE
```

执行 `mret` 后检查：

```text
回到 mepc
中断使能恢复
没有重复提交或漏提交
年轻 Store 没有错误写出
```

### 14.4 并发和优先级测试

```text
同步异常与 Timer IRQ 同时出现
Timer IRQ 在 D-cache miss 期间到达
Timer IRQ 在 MMIO Store 等待期间到达
Timer IRQ 在分支恢复期间到达
Timer IRQ 在另一个 Trap handler 中保持 pending
```

对当前核心，最重要的是精确副作用：

```text
不能重复执行已经提交的 Store
不能丢失尚未提交的指令
不能把中断错误归因于普通指令
```

### 14.5 裸机软件测试

在接 RT-Thread 前，写一个最小程序：

```text
1. 设置 mtvec
2. 设置第一次 mtimecmp
3. 开 MTIE/MIE
4. 主循环递增另一个变量
5. 每次 Timer ISR：
   - 设置下一次比较值
   - tick_count++
   - 每 1000 次通过 UART 打印一次
```

期望效果：

```text
timer tick 1000
timer tick 2000
timer tick 3000
```

并用板外秒表确认每 1000 Tick 约为 1 秒。

### 14.6 RT-Thread 系统测试

按顺序验证：

```text
rt_tick_get() 持续增加
rt_thread_mdelay(1000) 约等待 1 秒
两个同优先级线程能够时间片轮转
高优先级延时线程到期后能抢占
RT-Thread 软件定时器按期触发
FinSH 输入期间 Tick 不停止
运行 CoreMark 时 Tick 仍持续
```

CoreMark 运行时间较长时，Timer 中断会给测得性能带来少量 RTOS 开销。这是“在 RTOS 环境运行 CoreMark”的真实开销；如果比赛还要求纯 CPU 极限性能，可以另外保留裸机 CoreMark profile，但必须遵守现场规则。

---

## 15. 建议的断言和可观测信号

Timer RTL 可以加入仿真断言：

```text
非复位状态下，如果没有软件写 mtime，mtime 不得倒退
irq_timer_o 为 1 时，应满足 mtime >= mtimecmp
mtime < mtimecmp 时，irq_timer_o 最终应撤销
HXI response 背压时 payload 保持
一笔请求只产生一笔响应
非法寄存器访问必须返回 error
```

系统级建议导出调试信号：

```text
dbg_mtime[63:0]
dbg_mtimecmp[63:0]
dbg_mtip
dbg_take_timer_irq
dbg_mcause
dbg_mepc
dbg_rt_tick（软件可通过 UART 打印）
```

FPGA 上可以使用 ILA 观察：

```text
mtime 低位
mtimecmp 低位
irq_timer
CPU take_irq
mtvec redirect
```

这样能区分：

```text
Timer 没到点
Timer 已到点但 CPU 没使能
CPU 已进 Trap 但软件没识别 cause
软件识别了但没有正确写下一次 mtimecmp
```

---

## 16. 推荐实施顺序

### 阶段 1：冻结硬件—软件契约

书面确定：

```text
Timer 基地址与偏移
是否兼容 SiFive CLINT 偏移
TIMEBASE_FREQ
mtime/mtimecmp 的读写权限
对齐和 wstrb 规则
复位值
IRQ 电平语义
未实现 MSIP 时的行为
```

### 阶段 2：单独实现和验证 Machine Timer

暂时不接 CPU，用测试 Master 通过 HXI：

```text
读 mtime
写 mtimecmp
观察 irq
```

### 阶段 3：补齐 CPU 中断 CSR 和精确入口

用 Testbench 直接驱动 `irq_timer_i`，先不依赖真实 Timer。

验证：

```text
mie/mip/mstatus
mcause/mepc
mtvec
mret
同步异常优先级
```

### 阶段 4：Timer 与 CPU 联调

连接：

```text
Timer HXI Slave
Timer IRQ
CPU D-side MMIO
```

运行裸机中断测试。

### 阶段 5：接 RT-Thread Tick

完成：

```text
drv_timer.c
Trap 汇编入口
中断 enter/leave
rt_tick_increase()
RT_TICK_PER_SECOND
```

### 阶段 6：再接 FinSH 和 CoreMark

这样 CoreMark 出问题时，能够明确区分：

```text
CoreMark 移植问题
RT-Thread 调度问题
Timer/Trap 问题
总线/MMIO 问题
```

---

## 17. 当前项目最应该避免的几种错误

### 错误 1：把现有毫秒 Counter 改个名字就叫 CLINT

名字不会自动带来：

```text
mtimecmp
Timer IRQ
MTIP/MTIE
精确 Trap
```

### 错误 2：Timer 到点只发一个脉冲

中断可能被关闭、延迟或跨时钟域丢失。应采用 pending 电平。

### 错误 3：Timer 地址经过 D-cache

会读到旧时间，也可能让比较值更新延迟或乱序。

### 错误 4：只接 `irq_timer_i`，不实现 `mie/mip`

RT-Thread 无法按 RISC-V CSR 语义开关和识别中断。

### 错误 5：`mcause` 仍只有 5 位

同步异常 cause 7 与 Timer Interrupt cause 7 的区别在：

```text
mcause[31]
```

Timer 必须是：

```text
0x8000_0007
```

### 错误 6：RV32 直接把 64 位寄存器当一次访问

总线只有 32 位，必须处理高低半部和原子性窗口。

### 错误 7：照抄 RV64/S-mode/SBI BSP

当前目标是 RV32 M-mode，寄存器宽度、CSR、中断 cause 和 Timer 编程路径不同。

### 错误 8：把 Timer 中断接到 PLIC 后再进 CPU

Machine Timer 本身就是标准 local interrupt cause 7。UART/GPIO 等才通常进入外部中断控制器。

### 错误 9：Timer 频率只写在注释里

硬件分频参数、BSP 的 `TIMEBASE_FREQ` 和 `RT_TICK_PER_SECOND` 必须一致，并有测试验证一秒的实际 Tick 数。

---

## 18. 完成标准

不能以“Vivado 综合通过”作为 Timer 完成标准。至少应满足：

### RTL

- [ ] `mtime` 为 64 位固定频率单调计数；
- [ ] `mtimecmp` 为 64 位、软件可安全分两次 32 位访问；
- [ ] `irq_timer` 是 `mtime >= mtimecmp` 的电平 pending；
- [ ] Timer MMIO 走 HXI 明确完成或错误响应；
- [ ] Timer 地址强制 non-cacheable；
- [ ] 复位后不产生意外中断。

### CPU

- [ ] 有 `irq_timer_i`；
- [ ] 有 `mie.MTIE` 和 `mip.MTIP`；
- [ ] `mcause = 0x8000_0007`；
- [ ] 精确保存 `mepc`；
- [ ] 正确更新 `mstatus.MIE/MPIE`；
- [ ] `mret` 正确返回；
- [ ] 同步异常和异步中断优先级已验证。

### BSP/RT-Thread

- [ ] 能安全读取 RV32 `mtime`；
- [ ] 能安全写 RV32 `mtimecmp`；
- [ ] Timer ISR 会安排下一次截止时间；
- [ ] ISR 调用 `rt_interrupt_enter/rt_tick_increase/rt_interrupt_leave`；
- [ ] `rt_thread_mdelay(1000)` 实测约为 1 秒；
- [ ] 多线程时间片轮转和软件定时器可工作；
- [ ] FinSH/CoreMark 运行期间 Tick 不停止。

---

## 19. 最终架构判断

对当前项目，建议保留三种不同的“计时”能力：

```text
CPU 内部 cycle/instret/perf counters
    用于处理器性能、CoreMark 周期和 IPC 分析

SoC Machine Timer
    用于单调时间、Timer Interrupt 和 RT-Thread Tick

现有 START/STOP millisecond counter
    若比赛环境仍要求，可作为专用秒表保留
    若最终没有软件使用，也可以在系统稳定后删除
```

Machine Timer 在系统中的准确位置是：

```text
                    MMIO 配置路径
CPU LSU/D-cache ──→ HXI Crossbar ──→ Machine Timer
       ▲                                  │
       │                                  │ irq_timer
       └──────── CPU CSR/Trap 逻辑 ←──────┘
```

它处于 CPU 核和普通外设之间的硬件—软件契约边界：

- 向总线暴露寄存器；
- 向 CPU 暴露中断；
- 向 BSP 暴露固定频率时间基准；
- 向 RT-Thread 提供周期性 Tick 的硬件基础。

只要这四个边界写清楚，Timer 本身的 RTL 并不庞大。真正需要认真设计的是：64 位 RV32 访问、MMIO 顺序、精确中断和软硬件频率一致性。

---

## 20. 参考资料

- [RISC-V Privileged Architecture：Machine Timer (`mtime`/`mtimecmp`)](https://docs.riscv.org/reference/isa/priv/machine.html)
- [RISC-V ACLINT Specification（归档源码，包含 MTIMER、MSWI、SSWI 及与 SiFive CLINT 的关系）](https://github.com/riscvarchive/riscv-aclint/blob/main/riscv-aclint.adoc)
- [RISC-V Advanced Interrupt Architecture](https://docs.riscv.org/reference/aia/v1.0/intro.html)
- [RT-Thread 官方内核移植文档：OS Tick](https://github.com/RT-Thread/rt-thread/blob/master/documentation/3.kernel/kernel-porting/kernel-porting.md)
- [RT-Thread `rt_tick_increase()` 实现](https://github.com/RT-Thread/rt-thread/blob/master/src/clock.c)
- [RT-Thread RISC-V common64 Tick 示例](https://github.com/RT-Thread/rt-thread/blob/master/libcpu/risc-v/common64/tick.c)——用于理解 Tick 流程；当前 RV32 M-mode 项目不能直接照抄其中的 RV64/SBI/S-mode 实现。
