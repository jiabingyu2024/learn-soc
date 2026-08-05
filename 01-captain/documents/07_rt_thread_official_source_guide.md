# RT-Thread 官方源码导读

> 用户所说的“RTT-Three”在本文中按 **RT-Thread** 理解。  
> 官方仓库：[RT-Thread/rt-thread](https://github.com/RT-Thread/rt-thread)  
> 本地参考副本：[references/rt-thread-official](./references/rt-thread-official/)  
> 检查基准：官方 `master` 分支，提交 `1edc46a`，检出日期 2026-07-28。该提交源码中的版本宏为 `5.3.0`。

本文不按文件名逐项翻译 README，而是回答三个问题：

1. RT-Thread 仓库各目录为什么这样组织；
2. 从复位、启动、创建线程到发生调度，源码怎样连起来；
3. 自研 RV32 SoC 移植时，哪些代码可以使用，哪些代码必须由 BSP 和硬件共同完成。

## 1. RT-Thread 是什么

RT-Thread 是面向嵌入式系统的实时操作系统。官方仓库中的内容已经超过“一个线程调度内核”，还包括：

```text
实时内核
├─ 线程
├─ 调度
├─ 系统 Tick
├─ 软件定时器
├─ IPC
├─ 内存管理
└─ 内核对象

硬件适配
├─ 各 CPU 架构的上下文切换
├─ Trap/中断入口
├─ 板级初始化
├─ 外设驱动
└─ 链接和构建配置

上层组件
├─ FinSH/MSH 命令行
├─ 设备框架
├─ 文件系统
├─ 网络
├─ POSIX 兼容层
└─ 软件包生态
```

官方 README 将工程概括为内核层、组件与服务层、软件包三个层次，并把 `libcpu` 和 BSP 归入与芯片移植相关的部分。当前比赛项目首先需要内核、RV32 CPU port、BSP、UART、Timer 和可选的 FinSH；文件系统、网络和用户态功能可以后加。

### 1.1 Standard、Nano 和 Smart

在学习资料中经常看到三个名字：

| 名称 | 适合的系统 | 当前项目是否优先 |
| --- | --- | --- |
| RT-Thread Nano | 资源很小的 MCU，只保留紧凑内核 | 可用于最小实验，但不是本文主线 |
| RT-Thread Standard | 内核加可裁剪组件、设备框架、FinSH | **是** |
| RT-Thread Smart | 带用户态、MMU、进程等能力 | 否，当前自研 RV32 SoC 不需要 |

本文默认目标是：

```text
单核 RV32
M-mode
无 MMU
单地址空间
RT-Thread Standard
```

官方 `bsp/qemu-virt64-riscv` 同时覆盖更复杂的 RV64、S-mode、OpenSBI、MMU/Smart 等场景。它适合学习 BSP 目录和构建方式，不能整套复制到上述目标。

## 2. 官方仓库的顶层目录

本地稀疏克隆保留了主要学习目录。完整仓库顶层如下：

```text
rt-thread/
├─ src/                 内核实现
├─ include/             内核公共头文件
├─ libcpu/              CPU 架构适配
├─ bsp/                 不同芯片和开发板的 BSP
├─ components/          设备、FinSH、文件系统、网络等组件
├─ examples/            示例
├─ documentation/       工程文档
├─ tools/               SCons、Kconfig、工程生成等工具
├─ Kconfig              全局配置入口
├─ README_zh.md
├─ ChangeLog.md
└─ LICENSE
```

它不是一个直接在仓库根目录执行统一 `make` 的工程。通常进入某个 BSP 目录，由该 BSP 的 `SConstruct`、`rtconfig.py`、`rtconfig.h` 和 `link.lds` 组织一次具体构建。

### 2.1 哪些目录是“平台无关”的

大体上：

```text
src/ + include/
    平台无关内核

components/
    大部分平台无关，但依赖 BSP 提供设备

libcpu/
    CPU 架构相关

bsp/
    芯片、板卡和外设相关
```

边界并非绝对。例如设备框架是通用组件，具体 UART 寄存器读写仍在 BSP；通用 RV32 上下文切换位于 `libcpu`，而某个 CPU 核特殊的中断控制方法仍需 BSP 补充。

## 3. `src/`：RT-Thread 内核

当前 checkout 的 `src/` 包含：

```text
src/
├─ object.c
├─ thread.c
├─ scheduler_comm.c
├─ scheduler_up.c
├─ scheduler_mp.c
├─ clock.c
├─ timer.c
├─ ipc.c
├─ irq.c
├─ idle.c
├─ defunct.c
├─ components.c
├─ cpu_up.c
├─ cpu_mp.c
├─ mem.c
├─ memheap.c
├─ mempool.c
├─ slab.c
├─ signal.c
├─ kservice.c
└─ ...
```

### 3.1 `object.c`：统一的内核对象

RT-Thread 把线程、信号量、互斥量、事件、邮箱、消息队列、定时器和设备等看成不同类型的内核对象。它们拥有共同的对象头，例如名称、类型和链表节点。

统一对象机制带来几项能力：

- 以名称查找对象；
- 枚举线程或设备；
- 区分静态对象与动态对象；
- FinSH 中列出线程、信号量和设备；
- 采用类似的初始化、脱离、创建和删除接口。

接口名称常成对出现：

| 静态对象 | 动态对象 |
| --- | --- |
| `rt_thread_init()` | `rt_thread_create()` |
| `rt_thread_detach()` | `rt_thread_delete()` |
| `rt_sem_init()` | `rt_sem_create()` |
| `rt_sem_detach()` | `rt_sem_delete()` |

静态接口由调用者提供控制块和存储空间；动态接口从 heap 分配。初次移植阶段，可以先使用静态线程降低 heap 问题对调试的干扰。

### 3.2 `thread.c`：线程生命周期

`thread.c` 处理：

- 线程控制块初始化；
- 线程栈初始化；
- 创建和删除；
- 启动、挂起、恢复；
- 延时；
- 退出；
- 时间片和线程状态变化。

线程创建时，内核会调用架构层提供的 `rt_hw_stack_init()`。这个函数不在 `src/thread.c` 中实现，而在 `libcpu` 中实现，因为不同 CPU 的寄存器、Trap 返回指令和栈帧都不相同。

```text
rt_thread_create / rt_thread_init
          │
          ▼
分配或接收 TCB 与线程栈
          │
          ▼
rt_hw_stack_init()
          │
          ▼
在线程栈中构造初始寄存器现场
          │
          ▼
TCB->sp 指向该现场
```

调用 `rt_thread_startup()` 后，线程进入 ready 队列，但不等于立即执行。是否抢占当前线程由优先级、调度锁和中断上下文决定。

### 3.3 `scheduler_comm.c`、`scheduler_up.c` 和 `scheduler_mp.c`

当前仓库将调度代码按公共、单核和多核拆开：

| 文件 | 内容 |
| --- | --- |
| `scheduler_comm.c` | 调度公共逻辑和数据结构 |
| `scheduler_up.c` | UP，单处理器调度 |
| `scheduler_mp.c` | MP/SMP，多处理器调度 |

`src/SConscript` 会根据 `RT_USING_SMP` 选择：

```text
未启用 SMP：
  cpu_up.c
  scheduler_up.c

启用 SMP：
  cpu_mp.c
  scheduler_mp.c
```

自研单核 RV32 应先走 UP 路径。

调度器维护按优先级组织的 ready 队列。数字越小通常代表优先级越高。调度发生时，内核选出最高优先级的 ready 线程，然后调用 CPU port：

```text
rt_schedule()
  → 选择 from_thread 和 to_thread
  → 更新线程状态与 ready 队列
  → rt_hw_context_switch(...)

若在中断中决定切换：
  → rt_hw_context_switch_interrupt(...)
```

`rt_system_scheduler_start()` 用于第一次启动调度。当前单核实现选出最高优先级线程，将它设为 current thread，最后调用：

```c
rt_hw_context_switch_to(&to_thread->sp);
```

第一次调度不会返回到启动函数。

### 3.4 `clock.c`：系统 Tick

`clock.c` 维护 RT-Thread 的逻辑时间。硬件 Timer 中断应按固定频率调用：

```c
rt_tick_increase();
```

它会推进：

- 系统 Tick；
- 线程时间片；
- 延时/超时；
- 定时器检查；
- 必要的调度请求。

`RT_TICK_PER_SECOND` 在 `rtconfig.h` 中配置。例如官方 QEMU RISC-V BSP 当前配置为 100，表示一个 Tick 理论上为 10 ms。

RT-Thread 不知道自研 Timer 的 MMIO 地址，也不知道计数频率。BSP 必须完成：

```text
Timer 计数器配置
Timer compare 或 reload
Timer IRQ 使能
ISR 清除 pending
ISR 调用 rt_tick_increase()
```

### 3.5 `timer.c`：软件定时器

软件定时器建立在系统 Tick 之上，不是另一个硬件外设。

```text
硬件 Timer
  → 周期 IRQ
  → rt_tick_increase
  → RT-Thread 定时器管理
  → 到期回调
```

RT-Thread 支持硬定时器语义和软定时器线程语义。前者的回调可能处在 Tick/中断相关上下文，不适合阻塞；后者由软件定时器线程执行，可以承担稍多工作，但仍应避免长时间阻塞。

### 3.6 `ipc.c`：线程间同步与通信

当前 `ipc.c` 集中实现多种 IPC：

| IPC | 适合的用途 | 传递的数据 |
| --- | --- | --- |
| Semaphore | 事件计数、资源数量 | 计数值 |
| Mutex | 保护共享资源 | 所有权，支持优先级处理 |
| Event | 多个事件位的组合等待 | bit set |
| Mailbox | 传递固定机器字大小的值/指针 | `rt_ubase_t` |
| Message Queue | 传递固定最大长度的消息副本 | 数据块 |

阻塞式获取资源时，当前线程从 ready/running 转到 suspend，并挂入该 IPC 对象的等待队列。资源释放后，内核唤醒合适线程，再根据优先级判断是否调度。

ISR 中不能随意调用会阻塞的接口。通常允许发送/释放一类非阻塞操作，不允许在 ISR 里等待信号量、互斥量或消息。

### 3.7 `idle.c` 和 `defunct.c`

Idle 是系统总能运行的最低优先级线程。当没有其他 ready 线程时，CPU 运行 idle。此处可以放置低功耗等待、空闲 hook 或后台回收。

Defunct 相关代码处理已经退出、等待回收的动态线程资源。它解释了为什么线程入口函数返回后不应直接“落到未知地址”，而要进入 RTOS 提供的线程退出路径。

### 3.8 内存管理文件

RT-Thread 提供多种可裁剪的内存管理方式：

| 文件 | 用途 |
| --- | --- |
| `mem.c` | small memory allocator |
| `memheap.c` | 可管理一块或多块 heap 区域 |
| `mempool.c` | 固定大小内存块池 |
| `slab.c` | slab 风格分配器 |

`src/SConscript` 根据 Kconfig/`rtconfig.h` 选择真正参与编译的实现。不是把 `src/*.c` 无条件全部编译。

自研 SoC 的 BSP 还要提供合法 heap 边界。边界通常来自链接脚本符号：

```text
heap_begin = .bss 结束后对齐地址
heap_end   = RAM 顶部减去保留栈或其他区域
```

若 RAM 只有几十或几百 KiB，应先画清主栈、线程栈、heap、`.bss` 和测试数据之间的关系。

### 3.9 `components.c`：启动和自动初始化

`components.c` 包含两项很重要的机制：

1. RT-Thread 的系统启动序列；
2. 组件自动初始化表。

当前源码中的启动主线是：

```text
entry()
  → rtthread_startup()
      → 关闭本地中断
      → rt_hw_board_init()
      → 输出版本
      → rt_system_timer_init()
      → rt_system_scheduler_init()
      → 可选：signal 初始化
      → rt_application_init()
          → 创建并启动 main 线程
      → rt_system_timer_thread_init()
      → rt_thread_idle_init()
      → rt_thread_defunct_init()
      → rt_system_scheduler_start()
```

调度器启动后，main 线程最终执行：

```text
main_thread_entry
  → rt_components_init()
  → 用户 main()
```

这意味着用户 `main()` 通常已经在线程上下文中运行。它不是复位后的第一段 C 代码。

## 4. `include/`：内核对外接口和数据结构

当前主要头文件：

```text
include/
├─ rtdef.h
├─ rtthread.h
├─ rthw.h
├─ rtsched.h
├─ rtservice.h
├─ rtatomic.h
├─ rtcompiler.h
├─ rtdbg.h
└─ ...
```

### 4.1 `rtdef.h`

这里定义：

- RT-Thread 版本；
- 基础类型；
- 内核对象类型；
- TCB、Timer、IPC 等结构；
- 配置相关宏；
- 自动初始化宏；
- section/对齐相关定义。

阅读源码时，看到 `rt_thread_t`、`rt_sem_t` 等类型，应回到这里看它们是指针别名还是结构类型。

当前 checkout 的版本宏是：

```text
RT_VERSION_MAJOR = 5
RT_VERSION_MINOR = 3
RT_VERSION_PATCH = 0
```

它代表当前 `master` 源码状态，不等于建议比赛项目追随未冻结 master。正式集成时应固定 tag 或 commit。

### 4.2 `rtthread.h`

这是应用最常包含的内核 API 头文件，包含：

- 线程 API；
- Tick 和定时器；
- IPC；
- 内存；
- 内核服务。

### 4.3 `rthw.h`

`rthw.h` 集中声明硬件/CPU port 需要提供的接口，例如：

- 中断控制；
- 上下文切换；
- 栈初始化；
- 中断安装和屏蔽；
- CPU 相关 hook。

移植过程中出现未定义符号时，先判断该符号属于：

```text
内核已经实现
libcpu 应实现
BSP 应实现
某个可选组件开启后才实现
```

## 5. 自动初始化机制

RT-Thread 允许驱动和组件用宏注册初始化函数：

```text
INIT_BOARD_EXPORT
INIT_CORE_EXPORT
INIT_SUBSYS_EXPORT
INIT_PLATFORM_EXPORT
INIT_PREV_EXPORT
INIT_DEVICE_EXPORT
INIT_COMPONENT_EXPORT
INIT_ENV_EXPORT
INIT_APP_EXPORT
```

这些宏并不是在编译时直接调用函数。它们把函数指针放到带编号的特殊 section 中，例如 `.rti_fn.*`。链接脚本使用：

```text
__rt_init_start
KEEP(*(SORT(.rti_fn*)))
__rt_init_end
```

把这些条目保留并排序。运行时 `rt_components_init()` 遍历该范围并调用函数。

```text
驱动源文件
  → INIT_DEVICE_EXPORT(uart_init)
  → 函数指针进入特殊 section
  → link.lds KEEP
  → __rt_init_start ... __rt_init_end
  → 启动阶段逐项调用
```

因此链接脚本漏掉 `.rti_fn*` 或没有 `KEEP` 时，驱动源码虽然参与编译，初始化函数仍可能被 `--gc-sections` 删除。

自研 BSP 可选择：

- 在 `rt_hw_board_init()` 中显式调用 UART、Timer 初始化；
- 使用 RT-Thread 自动初始化宏；
- 两者结合，但要避免同一设备初始化两次。

## 6. `libcpu/`：CPU 架构移植

`libcpu` 不负责具体开发板的 UART 或 Timer 寄存器。它负责 CPU 架构相关、多个 SoC 可以复用的部分。

RISC-V 目录当前包含：

```text
libcpu/risc-v/
├─ common/      RV32 通用移植
├─ common64/    RV64 通用移植及 MMU/SBI 等
├─ rv64/
├─ virt64/
├─ t-head/
└─ vector/
```

当前自研 RV32 M-mode 项目应先研究：

[libcpu/risc-v/common](./references/rt-thread-official/libcpu/risc-v/common/)

不要从 `common64/startup_gcc.S` 或 `virt64` 直接移植。官方 QEMU RV64 代码使用 `stvec`、`sstatus`、SBI、PLIC、MMU 等机制，而目标 CPU 预计使用 `mtvec`、`mstatus` 和 M-mode 中断。

### 6.1 RV32 `common/` 文件

| 文件 | 职责 | 移植时的处理 |
| --- | --- | --- |
| `context_gcc.S` | 开关中断、首次切换、线程间切换、中断后切换 | 重点核对，可复用后适配 |
| `interrupt_gcc.S` | `SW_handler`，保存和恢复现场 | 核对 Trap 入口和栈帧 |
| `cpuport.c` | 初始线程栈、切换请求、weak hook | 通常复用并补 BSP hook |
| `cpuport.h` | 寄存器宽度、load/store 和相关宏 | 与 RV32/编译选项一致 |
| `rt_hw_stack_frame.h` | C 视角的完整线程/Trap 栈帧 | 必须与汇编偏移逐项一致 |
| `trap_common.c` | 通用中断注册和分发 | 可采用，也可接已有中断框架 |
| `riscv-ops.h` | CSR 操作 | 核对 CPU 支持 |
| `atomic_riscv.c` | 硬件原子支持 | 只在指令集与配置匹配时启用 |
| `readme.md` | 官方 RV32 移植说明 | 建议全文阅读 |

### 6.2 初始线程栈

`cpuport.c` 中的 `rt_hw_stack_init()` 会在新线程栈上放置一个 `rt_hw_stack_frame`：

```text
epc       = 线程入口 tentry
ra        = 线程退出处理 texit
a0        = 线程入口参数
mstatus   = 适合 mret 启动线程的初始状态
其他寄存器 = 调试填充值
```

当前源码在未启用 FPU 时把 `mstatus` 初始化为 `0x1880`，其意图是：

- 返回到 M-mode；
- Trap 返回后中断使能进入合适状态。

不要只因为这个常量来自官方代码就跳过核对。自研 CPU 必须正确实现相关 `mstatus` 字段和 `mret` 语义；若 CSR 位实现不同或存在定制中断机制，需按实际架构检查。

### 6.3 栈帧必须三处一致

需要一致的三处是：

```text
rt_hw_stack_frame.h
      ↕ 字段顺序和宽度
interrupt_gcc.S
      ↕ save/restore 偏移
context_gcc.S
```

还要与编译器 ABI 一致：

- RV32 使用 32 位寄存器宽度；
- RV32E 少寄存器，需要走条件分支；
- 启用 FPU 后要考虑浮点上下文；
- 栈对齐满足 ABI；
- `-march/-mabi` 与 CPU 实现匹配。

### 6.4 三种上下文切换入口

通用概念如下：

| 接口 | 使用场景 |
| --- | --- |
| `rt_hw_context_switch_to()` | 第一次启动调度，没有旧线程需要保存 |
| `rt_hw_context_switch()` | 线程上下文主动切换 |
| `rt_hw_context_switch_interrupt()` | ISR 结束时切换到另一线程 |

中断上下文切换不能简单地在 ISR 中直接调用普通线程切换汇编。此时已有一份 Trap 栈帧，切换动作常延迟到统一中断返回路径完成。

### 6.5 weak 接口意味着什么

RV32 `cpuport.c` 中存在 weak 默认实现，例如：

```text
rt_trigger_software_interrupt()
rt_hw_do_after_save_above()
rt_hw_context_switch_interrupt() 的部分路径
```

weak 不是“已经完整实现”。它允许 BSP 提供同名强符号覆盖默认实现。官方 `common/readme.md` 要求根据向量/非向量中断方式补齐相应接口。

### 6.6 通用中断分发

`trap_common.c` 提供：

```text
rt_hw_interrupt_init()
rt_hw_interrupt_install()
rt_rv32_system_irq_handler()
```

它维护向量号到处理函数的映射。是否使用这套分发取决于 SoC：

```text
方案 A：统一 mtvec 入口
  → 读取 mcause
  → rt_rv32_system_irq_handler
  → 查表执行 ISR

方案 B：硬件向量中断
  → 不同中断直接进入不同入口
  → BSP 适配软件中断和 RTOS 切换
```

若自研 CPU 使用标准 RISC-V direct `mtvec`，方案 A 较容易先跑通。

## 7. `bsp/`：板级支持包

BSP 把通用内核和通用 CPU port 连接到一个具体 SoC。它通常要说明：

```text
CPU 和架构
编译器与 ISA/ABI
内存地址
启动入口
链接布局
系统时钟
Timer
中断控制器
UART console
heap
应用入口
构建和配置
```

### 7.1 官方 QEMU RISC-V BSP 的目录

[bsp/qemu-virt64-riscv](./references/rt-thread-official/bsp/qemu-virt64-riscv/) 当前包含：

```text
bsp/qemu-virt64-riscv/
├─ applications/
│  ├─ main.c
│  └─ SConscript
├─ driver/
│  ├─ board.c
│  ├─ board.h
│  ├─ drv_uart.c
│  ├─ drv_virtio.c
│  ├─ virt.h
│  ├─ Kconfig
│  └─ SConscript
├─ SConstruct
├─ SConscript
├─ Kconfig
├─ rtconfig.h
├─ rtconfig.py
├─ link.lds
├─ link_smart.lds
├─ run.sh
└─ QEMU 调试脚本
```

它是很好的 BSP 文件组织参考，同时也是不适合直接复制的硬件实现参考：

| 可以参考 | 不应照搬 |
| --- | --- |
| 目录层次 | RV64 编译选项 |
| SCons 组织 | `0x80200000` 内存布局 |
| `board.c` 初始化顺序 | OpenSBI/S-mode |
| UART 驱动如何注册 | QEMU 16550 UART 地址 |
| `link.lds` 中 RT-Thread section | PLIC、VirtIO、MMU |
| `applications/main.c` | RV64 FPU/`lp64` ABI |

### 7.2 `board.c`

QEMU BSP 的 `rt_hw_board_init()` 当前按条件完成：

```text
可选 MMU 和页分配
  → heap
  → PLIC
  → 中断系统
  → UART
  → console 设备
  → Tick
  → 可选 SMP/IPI
  → board 级组件初始化
```

自研 SoC 可以缩减为：

```text
void rt_hw_board_init(void)
{
    rt_system_heap_init(heap_begin, heap_end);
    rt_hw_interrupt_init();
    rt_hw_uart_init();
    rt_console_set_device("uart0");
    rt_hw_tick_init();
}
```

这段只是职责示意。实际顺序还要保证在第一次 `rt_kprintf()` 前 UART 已经可用，在打开全局中断前 Trap 入口、Timer 和中断控制器已经配置好。

### 7.3 `applications/main.c`

官方 QEMU BSP 的 `main()` 只打印一行文本并返回。它仍然能运行，是因为 RT-Thread 在启动时创建 main 线程，并在线程入口中调用用户 `main()`。

自研 BSP 第一版的 `main.c` 建议也保持很小：

```text
打印启动信息
创建两个静态线程
线程周期输出或翻转 GPIO
主线程进入循环或正常返回
```

CoreMark、FinSH 和复杂组件在基础调度稳定后加入。

## 8. 设备框架和 UART

RT-Thread 的设备框架提供统一的：

```text
find
open
close
read
write
control
callback
```

UART 驱动通常分成两层：

```text
RT-Thread serial framework
  components/drivers/serial/
  components/drivers/include/drivers/dev_serial*.h

BSP low-level UART driver
  drv_uart.c
  ├─ configure
  ├─ control
  ├─ putc
  ├─ getc
  └─ ISR
```

### 8.1 以 QEMU `drv_uart.c` 为例

官方实现大致完成：

1. 填写 `rt_uart_ops`；
2. 配置 UART 基地址和中断号；
3. 调用 `rt_hw_serial_register()` 注册 `"uart0"`；
4. 安装 UART ISR；
5. 解除对应中断屏蔽；
6. 收到字符时调用 `rt_hw_serial_isr()` 通知 serial framework。

发送路径：

```text
rt_kprintf
  → console device
  → serial framework
  → BSP _uart_putc
  → UART status/data MMIO
  → RTL UART
```

接收路径：

```text
UART RX 事件
  → RTL UART pending
  → CPU external interrupt
  → BSP UART ISR
  → rt_hw_serial_isr
  → serial framework 唤醒接收等待
  → FinSH 读到字符
```

### 8.2 第一阶段可以不用完整设备框架吗

可以。裸机和移植早期可先实现：

```c
void rt_hw_console_output(const char *str);
```

用轮询 UART 验证日志。等线程和 Tick 稳定后，再接入 serial framework、RX 中断和 FinSH。

如果启用了 `RT_USING_CONSOLE` 并配置 `RT_CONSOLE_DEVICE_NAME "uart0"`，则需要确保名为 `uart0` 的设备在 console 设置前已经注册。

## 9. FinSH 与 MSH

FinSH 是 RT-Thread 的命令行组件；MSH 是常用的 shell 风格界面。相关源码位于：

[components/finsh](./references/rt-thread-official/components/finsh/)

当前目录主要包含：

```text
cmd.c          内置命令
msh.c          MSH 命令和执行
msh_parse.c    命令解析
shell.c        shell 线程和输入循环
finsh.h        导出宏和符号结构
Kconfig
SConscript
```

### 9.1 一个命令怎样出现

应用中写：

```c
static int led(int argc, char **argv)
{
    /* 控制 GPIO */
    return 0;
}

MSH_CMD_EXPORT(led, control led);
```

宏会生成一个命令描述条目，并把它放入 `FSymTab` section。链接脚本需要：

```text
__fsymtab_start
KEEP(*(FSymTab))
__fsymtab_end
```

shell 初始化时读取该范围，建立命令表。

```text
函数 + MSH_CMD_EXPORT
  → FSymTab
  → link.lds KEEP
  → FinSH 扫描
  → 用户在终端输入 led
  → shell 调用函数
```

命令没有出现时，依次检查：

- `RT_USING_FINSH`；
- `FINSH_USING_MSH`；
- `components/finsh` 是否参与构建；
- 源文件是否参与构建；
- `MSH_CMD_EXPORT` 是否启用；
- `link.lds` 是否保留 `FSymTab`；
- shell 线程是否启动；
- UART RX 和 console 是否工作。

## 10. Kconfig、SCons 和链接脚本

RT-Thread 官方工程常同时出现：

```text
Kconfig
.config
rtconfig.h
rtconfig.py
SConstruct
SConscript
link.lds
```

它们各自解决不同问题。

### 10.1 `Kconfig`

Kconfig 描述“有哪些选项、依赖关系是什么”。仓库根 `Kconfig` 会继续引入：

```text
src/Kconfig
libcpu/Kconfig
components/Kconfig
utest Kconfig
```

BSP 自己的 Kconfig 再选择架构并添加板级选项。

### 10.2 `.config` 和 `rtconfig.h`

菜单配置的结果通常先保存到 `.config`，再生成 C/汇编可见的 `rtconfig.h`。

`rtconfig.h` 会包含：

```text
线程名称长度
对齐
最大优先级
Tick 频率
heap
console
设备框架
FinSH
架构
SMP/Smart 等配置
```

源码中的 `#ifdef RT_USING_*` 依赖这些宏。

### 10.3 `rtconfig.py`

它描述工具链：

- GCC 前缀；
- CC/AS/AR/LD/objcopy/objdump；
- `-march`；
- `-mabi`；
- 优化和调试选项；
- 链接参数；
- 生成 `.bin` 和反汇编的后处理。

QEMU BSP 当前使用 RV64 `-march=rv64imafdc -mabi=lp64`。自研 RV32 必须改成与 CPU 完全一致的选项。

### 10.4 `SConstruct`

一个 BSP 的 `SConstruct` 是本次构建入口。QEMU BSP 中它：

```text
导入 rtconfig.py
  → 把 RTT_ROOT/tools 加入 Python 路径
  → 创建编译环境
  → PrepareBuilding
  → 选择 CPU/ARCH
  → 生成链接脚本包含项
  → DoBuilding
```

### 10.5 `SConscript`

每个目录的 `SConscript` 声明：

- 哪些源码属于该 group；
- include path；
- 依赖哪些 Kconfig 宏；
- 是否递归进入子目录；
- 局部编译选项。

例如 `src/SConscript` 会根据内存算法、signal、SMP 等选项移除不需要的源文件；`libcpu/risc-v/common/SConscript` 会在未启用硬件原子支持时移除 `atomic_riscv.c`。

### 10.6 `link.lds`

除了普通 `.text/.data/.bss/stack`，RT-Thread 链接脚本还要保留：

```text
FSymTab / VSymTab       FinSH 命令和变量
.rti_fn*                自动初始化函数
UtestTcTab              单元测试表
构造/析构表             使用 C++ 或 init_array 时
```

自研 BSP 可以参考官方链接脚本的 section 组织，内存地址和长度必须按自己的 SoC 重写。

## 11. 官方 QEMU BSP 怎样构建

本地参考目录：

[bsp/qemu-virt64-riscv](./references/rt-thread-official/bsp/qemu-virt64-riscv/)

官方 README 给出的典型流程为：

```text
进入 BSP 目录
  → 设置 RTT_CC_PREFIX 和 RTT_EXEC_PATH
  → scons --menuconfig
  → scons
  → 生成 rtthread.elf
  → objcopy 生成 rtthread.bin
  → run.sh 启动 QEMU
```

该 BSP 可用于观察 RT-Thread 正常启动后的表现。建议练习：

1. 保持默认配置构建运行；
2. 查看 `rtthread.map`；
3. 用 `objdump` 找 `entry`、`rtthread_startup` 和 `main`；
4. 修改 `applications/main.c` 创建两个线程；
5. 用 `list_thread`/`ps` 查看线程；
6. 跟踪 `rt_thread_mdelay()` 到 Tick 唤醒；
7. 再回到 RV32 `common` 阅读上下文切换。

它不是自研 RV32 SoC 的软件镜像。QEMU `virt` 的地址、UART、PLIC、SBI 和特权模式均与目标平台不同。

## 12. 自研 RV32 BSP 应该长什么样

推荐在实际工程中建立独立 BSP，而不是修改官方 QEMU BSP：

```text
sw/
├─ rt-thread/                    固定 tag/commit 的上游源码
└─ bsp/
   └─ superscalar-rv32/
      ├─ applications/
      │  ├─ main.c
      │  └─ SConscript
      ├─ board/
      │  ├─ board.c
      │  ├─ board.h
      │  ├─ trap_gcc.S
      │  ├─ drv_uart.c
      │  ├─ drv_timer.c
      │  └─ SConscript
      ├─ Kconfig
      ├─ SConstruct
      ├─ SConscript
      ├─ rtconfig.h
      ├─ rtconfig.py
      ├─ link.lds
      └─ README.md
```

也可以把 BSP 放进官方仓库的 `bsp/` 下。比赛项目更适合把上游内核和自己的 BSP 分清，方便升级、对比和提交源码。

### 12.1 可以直接引用的上游部分

第一版通常需要：

```text
src/
include/
libcpu/risc-v/common/       经过接口核对后使用
components/finsh/           可选，后加
components/drivers/core/    使用设备框架时
components/drivers/serial/  使用 serial framework 时
components/drivers/include/
tools/                      使用官方 SCons 构建时
```

### 12.2 必须自己提供或修改的部分

```text
start.S / reset entry
link.lds 中的真实 ROM/RAM
rtconfig.py 的 -march/-mabi
rtconfig.h / Kconfig
rt_hw_board_init
heap begin/end
UART MMIO driver
Timer 初始化和 ISR
中断控制器
mtvec/Trap 接入
weak hook 的平台实现
固件镜像生成与加载
```

### 12.3 第一阶段配置

建议先启用：

```text
RT_USING_USER_MAIN
线程和单核调度
Timer/Tick
静态线程
基本 console
```

随后启用：

```text
RT_USING_HEAP
RT_USING_DEVICE
serial framework
RT_USING_FINSH
FINSH_USING_MSH
UART RX interrupt
```

先关闭：

```text
RT_USING_SMP
RT_USING_SMART
MMU
文件系统
网络
复杂 POSIX
VirtIO
FPU/Vector（除非硬件已完整支持）
```

## 13. 移植接口清单

### 13.1 CPU/Trap

- [ ] `mtvec` 指向正确入口；
- [ ] Trap 入口保存和恢复全部所需寄存器；
- [ ] `mepc/mstatus` 正确保存恢复；
- [ ] `mret` 行为符合特权规范；
- [ ] `rt_hw_interrupt_disable()` 返回旧状态；
- [ ] `rt_hw_interrupt_enable(level)` 恢复旧状态；
- [ ] `rt_hw_stack_init()` 与汇编栈帧一致；
- [ ] 三种 context switch 路径可用；
- [ ] 中断中发起调度的机制明确；
- [ ] 异常能打印 `mcause/mepc/mtval`。

### 13.2 Timer

- [ ] Timer 时钟频率明确；
- [ ] `RT_TICK_PER_SECOND` 与 compare/reload 一致；
- [ ] pending 能可靠保持和清除；
- [ ] ISR 调用一次 `rt_tick_increase()`；
- [ ] 不会重复进入同一未清除中断；
- [ ] 长时间运行无 Tick 漂移或丢失。

### 13.3 UART

- [ ] 轮询发送先通过；
- [ ] 波特率与真实外设时钟一致；
- [ ] console 输出可在早期启动使用；
- [ ] 注册名与 `RT_CONSOLE_DEVICE_NAME` 一致；
- [ ] RX 中断和 ISR 后加；
- [ ] FinSH 输入不丢字符。

### 13.4 链接和内存

- [ ] reset vector、ELF entry、ROM 地址一致；
- [ ] `.text/.rodata` 落在可执行存储器；
- [ ] `.data/.bss/heap/stack` 落在可写 RAM；
- [ ] heap 不与栈重叠；
- [ ] 保留 `.rti_fn*`；
- [ ] 使用 FinSH 时保留 `FSymTab/VSymTab`；
- [ ] 生成 map 并检查 section；
- [ ] 仿真与 FPGA 的 RAM 延迟一致。

## 14. 推荐的源码阅读顺序

### 第一遍：知道每层在哪里

```text
README_zh.md
  → src/SConscript
  → bsp/qemu-virt64-riscv/SConstruct
  → bsp/qemu-virt64-riscv/rtconfig.py
  → bsp/qemu-virt64-riscv/link.lds
```

目标是理解一次构建怎样选中内核、CPU port、BSP、组件和应用。

### 第二遍：启动和第一次调度

```text
架构 startup
  → src/components.c: entry / rtthread_startup
  → rt_hw_board_init
  → rt_application_init
  → rt_system_scheduler_start
  → libcpu context switch
  → main_thread_entry
  → 用户 main
```

要求能画出调用图，并标出每一步运行在启动栈还是线程栈。

### 第三遍：线程和 Tick

```text
src/thread.c
  → rt_hw_stack_init
  → scheduler_up.c
  → clock.c
  → timer.c
```

要求解释：

- 新线程的 `sp` 如何产生；
- 最高优先级线程如何选出；
- `rt_thread_mdelay()` 为什么不忙等；
- Timer IRQ 如何让线程重新 ready。

### 第四遍：RV32 上下文

```text
libcpu/risc-v/common/readme.md
  → rt_hw_stack_frame.h
  → cpuport.c
  → context_gcc.S
  → interrupt_gcc.S
  → trap_common.c
```

为每个寄存器画出栈偏移，核对 `mstatus/mepc`。

### 第五遍：UART 和 FinSH

```text
QEMU drv_uart.c
  → components/drivers/serial
  → console
  → components/finsh/shell.c
  → finsh.h 导出宏
  → link.lds FSymTab
```

要求能从 UART RX 引脚一直追到 shell 命令函数。

## 15. 常见误区

### 15.1 把 `libcpu/risc-v` 整个目录都加入编译

这里同时有 RV32、RV64、平台定制和 vector 代码。应由 Kconfig/SConscript 选择正确子目录，否则会出现重复符号、错误 CSR 和错误 ABI。

### 15.2 把 QEMU RV64 BSP 当作 RV32 模板直接复制

它使用 RV64、S-mode、OpenSBI、PLIC、MMU/Smart 相关代码，存储器从 `0x80200000` 开始。只能参考组织和 RT-Thread section。

### 15.3 看到 weak 函数就认为已有实现

weak 默认函数可能只是空操作或死循环。平台需要覆盖的函数若未覆盖，链接仍可能成功，运行时却卡住。

### 15.4 线程能创建就认为 context port 正确

线程创建只构造栈。真正验证需要：

- 第一次切换；
- 线程间主动切换；
- Timer ISR 中抢占切换；
- 长时间运行；
- 不同优化等级；
- 足够深的函数调用；
- 必要时浮点/原子场景。

### 15.5 只实现 UART 输出就打开 FinSH

FinSH 还需要 UART 输入、RX 中断或可用的阻塞读取、shell 线程和符号表 section。只有 `rt_kprintf()` 不代表 shell 可用。

### 15.6 忘记链接脚本中的 RT-Thread 表

`--gc-sections` 会删除“C 调用图中看似未使用”的导出项。自动初始化、FinSH 命令和测试表必须通过 `KEEP` 保留。

### 15.7 直接追官方 `master`

`master` 会变化。比赛项目应记录：

```text
RT-Thread tag 或 commit
本地修改
编译器版本
-march/-mabi
rtconfig.h
```

当前本地参考 clone 用于学习，不应在没有评审的情况下自动覆盖比赛工程中的固定版本。

## 16. 本地参考仓库的使用方法

克隆位置：

[E:/Resources/03_competitions/26_03_jcs/2608round/learn/references/rt-thread-official](./references/rt-thread-official/)

这是浅层、稀疏 clone：

```text
当前展开：
  src
  include
  libcpu/risc-v
  bsp/qemu-virt64-riscv
  components/drivers
  components/finsh
  documentation
```

查看版本：

```powershell
git -C .\references\rt-thread-official rev-parse --short HEAD
git -C .\references\rt-thread-official log -1
```

展开更多目录，例如文件系统和网络：

```powershell
git -C .\references\rt-thread-official sparse-checkout add components/dfs components/net
```

更新浅层参考：

```powershell
git -C .\references\rt-thread-official pull --ff-only
```

更新前要注意：本文按 `1edc46a` 写成，更新后目录和接口可能变化。正式移植最好从一个 tag 或确定 commit 建立独立副本。

## 17. 当前项目真正需要从官方仓库取得什么

最小 RT-Thread bring-up 可以归结为：

```text
官方提供：
  src/
  include/
  RV32 common port 的主体

项目实现：
  启动和链接
  UART
  Timer
  Trap/IRQ 接入
  heap 和内存布局
  构建配置

可选官方组件：
  serial framework
  FinSH/MSH
```

RT-Thread 内核不会替 SoC 实现中断控制器和 UART。SoC 也不会替 RT-Thread知道线程优先级和等待队列。它们的交界集中在：

```text
rt_hw_stack_init
rt_hw_context_switch*
rt_hw_interrupt_disable/enable
rt_hw_board_init
rt_hw_tick_init
Timer ISR → rt_tick_increase
UART driver / console
link.lds
```

先让这组接口形成可验证闭环，再添加设备框架、FinSH 和 CoreMark。

## 18. 学完本文后的验收问题

能够回答下列问题，说明已经具备阅读和裁剪官方仓库的基本能力：

1. 为什么用户 `main()` 不是复位后的第一段代码？
2. `rt_thread_create()` 为什么要调用 `rt_hw_stack_init()`？
3. 第一次调度和普通线程切换有什么区别？
4. Timer RTL 与 `rt_tick_increase()` 之间还缺哪些 BSP 逻辑？
5. `libcpu` 和 BSP 的职责边界在哪里？
6. 为什么 QEMU RV64 的 `startup_gcc.S` 不能直接给 RV32 M-mode 用？
7. `rtconfig.h`、`rtconfig.py`、Kconfig 和 SConscript 分别控制什么？
8. 为什么 FinSH 命令需要链接脚本 `KEEP`？
9. weak port 函数未覆盖时会出现什么现象？
10. 一次 UART RX 怎样唤醒 FinSH shell 线程？
11. `.text/.data/.bss/heap/stack` 怎样与 SoC 地址表对应？
12. 如何固定并记录比赛使用的 RT-Thread 版本？

## 19. 相关资料

- [RT-Thread 官方 GitHub 仓库](https://github.com/RT-Thread/rt-thread)
- [官方 RV32 common port](https://github.com/RT-Thread/rt-thread/tree/master/libcpu/risc-v/common)
- [官方 QEMU RISC-V BSP](https://github.com/RT-Thread/rt-thread/tree/master/bsp/qemu-virt64-riscv)
- [RT-Thread Kernel Porting](https://www.rt-thread.io/document/site/programming-manual/porting/porting/)
- [本系列：RT-Thread 层](./02_rt_thread_layer.md)
- [本系列：BSP 与软件移植层](./03_bsp_porting_layer.md)
- [本系列：SoC 与 RTOS 移植学习指南](./06_soc_os_porting_learning_guide.md)

