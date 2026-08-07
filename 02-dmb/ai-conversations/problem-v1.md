结论先说：v1.5 板上无法启动的根因不是 CPU 核、UART、超频，也不是 BRAM 内容或双口合并本身，而是新加入的“专用流水取指旁路”没有实现完整、可自保持的请求/响应事务语义。

严格来说，v1.5 当次主要是在已有优化上选择 175 MHz 并归档；真正引入故障的取指架构改动始于 v1.1，在 v1.2 中进一步做了时序解耦，随后被 v1.3～v1.5 沿用。大家所说的“v1.5 修改”，实际是 v1.1～v1.5 的累计工作树。

## 一、为什么要进行这轮修改

最初诊断发现，CoreMark 的主要瓶颈是旧 HXI 取指通路。

良品结构是：

```text
CPU
  → hxi_instruction_adapter
  → HXI crossbar
  → memory_subsystem
  → generic_rom
  → Code BRAM
```

旧适配器每次只能存在一笔未完成请求：

1. CPU 发出 PC 请求；
2. 等待 HXI 接受；
3. 等待 ROM 返回；
4. 适配器把响应再寄存；
5. CPU 消费响应后才能开始下一笔取指。

实测仿真约为：

```text
CoreMark：944,997 cycles/iteration
IPC：约 0.317
CPI：约 3.15
```

约 95% 的执行时间可以由“三拍左右取到一条指令”解释。100 MHz 下运行 10000 轮约需 94.5 秒，即使只把频率提高到 200 MHz，仍约需 47 秒，无法满足 30 秒要求。

因此当时的优化方向是合理的：让 Code BRAM 每拍接收一个地址、下一拍返回指令，把顺序取指吞吐提高到接近每拍一条。

这一方向也符合 [plan.md](D:/jichuang_files/ai-conversations/plan.md) 中的性能分析。但该方案同时要求：

- 请求接受语义；
- 响应/skid buffer；
- 分支重定向的 epoch 或地址标签；
- 响应背压；
- 请求和响应的稳定保持。

v1.5 只实现了“直接 BRAM 端口”和“地址标签”，没有实现其余关键事务机制。

## 二、v1.5 累计做了哪些修改

### 1. 用专用流水适配器取代 HXI 取指

`cpu_subsystem` 中原来的：

```systemverilog
hxi_instruction_adapter
```

被替换为：

```systemverilog
pipelined_instruction_adapter
```

归档源码见：

[pipelined_instruction_adapter.sv](D:/jichuang_database/report/soc_dev_v1_work/v1.5_175MHz/soc_dev_v1.5_pipelined_instruction_adapter.sv)

CPU 指令侧不再连接 `instr_hxi`，而是新增：

```text
instr_req_valid
instr_req_addr

instr_rsp_valid
instr_rsp_addr
instr_rsp_data
instr_rsp_err
```

这些信号一路穿过：

```text
cpu_subsystem
→ soc_core
→ soc_top_generic / fpga_top
→ xilinx_code_mem_backend
→ generic_rom
```

### 2. 关闭原来的 HXI instruction master

`soc_core` 中的 `cpu_i_hxi` 被固定为空闲：

```systemverilog
cpu_i_hxi.req_valid = 0;
cpu_i_hxi.rsp_ready = 0;
```

因此取指完全绕过了已经验证过的：

```text
hxi_instruction_adapter
HXI crossbar
memory_subsystem
```

不过 D-side 仍可通过原 HXI 路径读取 Code ROM 中的 `.rodata` 和字符串常量。

### 3. 给 `generic_rom` 增加第二个同步读端口

`generic_rom` 原本只有 native/HXI 访问端口。修改后又加入独立指令端口：

```systemverilog
instr_req_valid_i
instr_req_addr_i
instr_rsp_valid_o
instr_rsp_addr_o
instr_rsp_data_o
instr_rsp_err_o
```

地址、有效位和错误位寄存一拍，数据也从 `storage[]` 同步读取。

v1.5 中，Vivado 最终把：

- 专用取指读端口；
- D-side 代码数据读端口；

合并成同一组真双口 Code BRAM。资源结果是：

```text
Code BRAM：16 RAMB36
Data BRAM：16 RAMB36
总计：32 RAMB36
```

这项变化后来一度被怀疑是故障原因，但后续实验已经排除。

### 4. 实现顺序预读 `PC+4`

v1.5 适配器的核心逻辑是：

```systemverilog
response_matches =
    fetch_rsp_valid &&
    fetch_rsp_addr == core_req_addr;

fetch_req_addr = core_req_addr;

if (response_matches)
    fetch_req_addr = core_req_addr + 4;
```

也就是说：

- 默认读取 CPU 当前 PC；
- 当返回数据恰好属于当前 PC 时，立即把下一次 BRAM 地址切换到 `PC+4`；
- 分支跳转后，如果返回地址不等于当前 PC，就把它作为陈旧顺序响应丢弃。

这样在理想顺序执行时可以形成：

```text
周期 N：返回 PC，同时提交 PC+4
周期 N+1：返回 PC+4，同时提交 PC+8
周期 N+2：返回 PC+8……
```

### 5. 为了时序，刻意移除了 `core_rsp_ready` 对预读地址的控制

适配器注释明确说明，不希望把 CPU stall/load/multiply 状态放进 BRAM 地址组合路径，因此即使 CPU 当前不能接收响应，只要地址匹配，也照样读取 `PC+4`。

这就是 v1.2“timing decoupled”改动的核心，随后被 v1.3～v1.5 沿用。

### 6. 配套性能与配置修改

累计工作树还包含：

- 软件 ISA 从 RV32I 提升到 `rv32im_zicsr_zicntr_zifencei`；
- CoreMark/RT-Thread 从 `-O2` 改为 `-O3`；
- 关闭没有使用的 F 执行单元；
- SoC、Timer、UART divisor 和 CoreMark tick 配置改为 175 MHz；
- MMCM 配置为：

```text
200 MHz × 4.375 ÷ 5 = 175 MHz
```

- Vivado 构建目录加入频率标签；
- WSL/Verilator 路径处理得到修正。

其中 ISA、O3、关闭 F 等修改不是启动故障根因：v1.9.0 保留了这些 CPU 侧修改，只恢复 HXI 取指，板子就重新启动了。

v1.5 的完整累计补丁见：

[v1.5 worktree patch](D:/jichuang_database/report/soc_dev_v1_work/v1.5_175MHz/soc_dev_v1.5_worktree.patch)

## 三、为什么这个设计在仿真中很快，但实板不能启动

v1.5 的仿真结果确实很好：

```text
CoreMark CRC：PASS
约 430,806 cycles/iteration
175 MHz 推算：24.6175 秒/10000 iterations
```

相比约 944,997 cycles/iteration，周期数减少约 54%。从性能方向看，流水取指达到了预期。

但这个适配器不是一个完整的 ready/valid 协议实现。

| 事务属性 | 良品 HXI 路径 | v1.5 专用旁路 |
|---|---|---|
| 请求接受 | `req_valid && req_ready` | 没有 `req_ready` |
| 未完成事务 | `outstanding_q` 明确记录 | 没有 |
| 请求地址记录 | 请求握手时锁存 | 没有 |
| 响应保存 | 一项响应寄存器保持 | 没有 |
| CPU 背压 | 响应保持到 CPU 接收 | 不保持 |
| 重定向处理 | 排空旧事务并比较已锁存地址 | 与 CPU 当前 PC 即时比较 |
| 下一地址推进 | 事务完成后推进 | 地址匹配就读 `PC+4`，不管 CPU 是否 ready |

最危险的一种情况是：

```text
当前返回地址 = PC
core_rsp_ready = 0，CPU 正在 stall
```

v1.5 此时仍然认为 `response_matches=1`，于是向 BRAM 发出 `PC+4`。下一拍：

```text
CPU 的 PC 仍是原 PC
BRAM 返回的是 PC+4
地址不匹配
core_rsp_valid 变成 0
```

然后适配器又把地址切回原 PC，形成：

```text
PC → PC+4 → PC → PC+4……
```

更严重的是，CPU 的 `hazard_unit` 又直接使用 `irom_valid` 控制 PC 停顿：

```systemverilog
o_stall_p_f = ... || !i_fetch_valid;
```

因此取指响应是否匹配，不只是一个“数据可用”信号，还会反过来影响 PC、stall、flush 和分支重定向。响应返回、CPU stall、分支 redirect 同周期出现时，v1.5 没有独立保存：

- 哪一笔请求真正被接受；
- 当前未完成请求属于哪个 PC；
- 哪个响应应保持到 CPU 接收；
- 重定向后哪个 epoch 的响应必须丢弃。

它依赖“当前 CPU PC、当前返回标签和下一拍 BRAM 地址恰好始终重新对齐”。这在理想 RTL 模型里可以工作，但不是一个完整、自保持、可背压的硬件协议。

这也正好违反了 `jichuang_files` 中既有设计文档规定的接口原则：

- [09_hxi_interconnect_bram_cache_architecture.md](D:/jichuang_files/documents/09_hxi_interconnect_bram_cache_architecture.md) 要求请求和响应都有 `valid/ready`，背压时 payload 必须保持；
- [14_rtl_internal_hierarchy_and_soc_interfaces.md](D:/jichuang_files/documents/14_rtl_internal_hierarchy_and_soc_interfaces.md) 建议 IF 接口包含 `if_req_ready`、response hold 和 epoch/tag；
- 文档推荐 CPU 子系统对 SoC 保持两组 HXI Master，而 v1.5 直接从 CPU 穿过 SoC/FPGA 顶层连接 BRAM，绕开了这一稳定边界。

## 四、为什么可以确定是这条旁路，而不是其他问题

排查过程形成了完整的 A/B 证据链。

### 1. 不是超频或时序违例

v1.5 的时序是：

```text
175 MHz
WNS -0.731 ns
TNS -380.724 ns
```

虽然在比赛允许范围内，但仍可能怀疑超频。

因此生成了同一故障架构的 100 MHz 时序干净版本 v1.7：

```text
WNS +1.237 ns
TNS 0
WHS +0.059 ns
THS 0
```

它在板上仍然完全没有 UART 输出。因此时序和超频被排除。

### 2. 不是 Code BRAM 真双口合并

v1.8.2 使用 `DONT_TOUCH` 和 `KEEP_HIERARCHY` 强制保留两个物理 Code ROM 副本：

```text
取指 ROM：16 RAMB36，单读口
代码数据 ROM：16 RAMB36，单读口
Data RAM：16 RAMB36
总计：48 RAMB36
```

时序也完全通过：

```text
WNS +0.931 ns
TNS 0
```

板上仍然不能启动。因此排除了：

- 两个逻辑 ROM 被合并；
- TDP BRAM A/B 端口冲突；
- BRAM 数量不足；
- 简单的 Code ROM 初始化副本问题。

所以负责人所说“问题一般在 SoC 的 BRAM 部分”在定位方向上接近：故障确实位于 SoC 到指令 BRAM 的访问边界；但并不是 BRAM 存储阵列或合并方式出错，而是 BRAM 前面的取指事务控制不完整。

### 3. 只恢复 HXI 取指后立即启动

v1.9.0 保留当前 CPU 核、RV32IM、O3、F 单元裁剪和其他外围修改，只把取指路径恢复成良品结构：

```text
CPU
→ hxi_instruction_adapter
→ HXI crossbar
→ memory_subsystem
→ generic_rom
```

结果：

```text
WSL RT-Thread/UART/msh：PASS
WNS +0.871 ns
TNS 0
板上：正常启动并进入 msh
```

因此故障随专用取指旁路加入而出现，随该旁路撤销而消失；CPU 核没有跟着回退。这是最有决定性的证据。

## 五、为什么 WSL 仿真没有发现

现有仿真只覆盖了一个非常理想的存储模型：

- ROM 固定一拍响应；
- 每个有效地址都被默认接受；
- 没有请求侧随机 `ready`；
- 没有响应侧随机背压；
- 没有验证响应在 CPU 不 ready 时必须保持；
- 没有系统性覆盖 stall、redirect、响应返回三者同周期组合；
- 没有针对请求稳定性、响应唯一性和 stale response 的协议断言；
- 没有门级或 post-route 取指接口仿真。

因此仿真证明的是：

> 在理想、确定性的一拍 ROM 模型下，v1.5 可以运行 RT-Thread 和 CoreMark。

它没有证明：

> v1.5 的取指接口具备完整硬件事务语义，并能在综合后的 FPGA BRAM 和真实 CPU stall/redirect 组合下可靠工作。

## 六、最终来龙去脉

完整因果链可以压缩为：

```text
CoreMark 约 95% 时间受三拍 HXI 取指限制
    ↓
决定给 Code BRAM 增加每拍读取的专用取指口
    ↓
v1.1 绕过 HXI，加入 pipelined_instruction_adapter
    ↓
v1.2 为缩短时序路径，进一步取消 core_rsp_ready 对预读地址的控制
    ↓
形成“地址匹配即读取 PC+4”的无 ready、无 outstanding、无响应保持旁路
    ↓
v1.3～v1.5 只继续提高/筛选频率，保留这一架构
    ↓
理想 WSL 模型中 CoreMark 降到约 430k cycles/iteration
    ↓
综合后板上取指事务失配，CPU 在启动早期即无法继续执行
    ↓
没有 UART 启动输出，Enter 自然无响应
    ↓
100 MHz 时序干净版仍失败，排除超频
    ↓
强制两个独立 BRAM 副本仍失败，排除 BRAM 合并
    ↓
恢复良品 HXI 取指、保留 CPU 修改后启动成功
    ↓
确认根因是专用流水取指旁路及其不完整事务语义
```

还有一个次要但独立的问题：不同频率构建时，Make 可能复用旧的软件对象，导致 100 MHz bit 打印 `clock=175000000 Hz`。这会使 CoreMark 秒数换算失真，但不会导致 CPU 完全无法启动，也不是 v1.5 无 UART 的根因。