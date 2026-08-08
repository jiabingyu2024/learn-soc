# 从分支预测优化到当前版本的系统修改总结

本文只记录当前工作区和当前分支中仍然保留、会影响系统行为或构建结果的修改。已经撤销的方案不列入当前系统配置。

## 1. 取指前端与分支预测

### GShare方向预测

- 保留独立的BTB和GShare方向预测结构。
- GShare历史长度设置为9位。
- PHT容量为512项，每项为2位饱和计数器。
- 预测索引使用PC低位与全局历史异或。
- 预测器采用同步读取，并在前端请求、响应和流水线epoch之间做匹配，避免重定向后的旧预测进入取指队列。

### BTB索引

- BTB容量为128项。
- BTB索引使用PC两段位异或折叠，降低简单低位索引造成的冲突。
- BTB仍只保存目标地址和分支类型，不改变总线协议。

### 小型循环预测器

- 增加16项循环预测表。
- 保存循环标签、推测的trip count、当前迭代计数和2位置信度。
- 只对稳定的向后条件分支进行覆盖式预测。
- 循环索引同样采用PC位折叠。
- 循环预测器不改变分支的实际执行和提交结果，低置信度时回退到GShare/BTB。

当前CoreMark 10轮测试中，分支预测准确率约为91%，分支预测优化没有修改外部总线。

## 2. 性能计数与仿真分析设施

### 硬件性能计数器

当前保留的计数器包括：

- cycle
- commit
- branch
- branch miss
- load/store
- DCache access/miss
- front/memory/muldiv/RAW stall

计数器只观察已有流水线事件，不参与PC选择、提交、异常或总线握手逻辑。计数器还保留了基本一致性断言，例如commit不超过cycle、branch miss不超过branch。

### 提交轨迹

仿真器保留 `--commit-profile` 选项，可以在性能窗口内导出：

```text
cycle, pc, instruction
```

该轨迹用于统计函数占比、循环回跳、指令组合和融合候选，不会改变被测程序的执行逻辑。

## 3. Zbkb支持

- RTL中启用 `CFG_ZBKB=1`。
- 软件编译选项启用：

```text
-march=rv32imfd_zicsr_zicntr_zifencei_zbkb
```

- 当前固件可以使用Zbkb指令，CoreMark CRC检查通过。
- Zbkb属于标准位操作扩展，不修改总线。

## 4. 编译器参数接口

在 `software/toolchain/common_flags.mk` 中保留了可选的：

```make
EXTRA_OPT_FLAGS
```

它允许扫描循环展开、调度、对齐等现有GCC参数，而不需要修改编译器源码。当前默认值为空，当前性能版本使用经过验证的默认O3配置。

## 5. 指令级融合：macc16和bfmacc16

这是当前版本新增的主要CoreMark专用优化，但融合粒度限制在单个矩阵元素运算，不是整个函数或整个矩阵模块。

### macc16

```text
macc16 acc, x, y
acc = acc + signed16(x) * signed16(y)
```

### bfmacc16

```text
p   = signed16(x) * signed16(y)
acc = acc + (((p >> 2) & 15) * ((p >> 5) & 127))
```

### RTL实现

- 自定义指令使用custom-0编码空间。
- 译码后进入现有bitmanip执行接口。
- `macc16/bfmacc16`使用目的寄存器作为旧累加值，因此目的寄存器同时是写回目的和第三个源操作数。
- Regfile增加第三个读端口。
- Scoreboard和operand resolver增加第三源的producer、ready、data和完成旁路检查，保证连续累加不会读到旧值。
- 融合计算已经改为三级流水：
  1. 16位有符号乘法
  2. 位域提取和小位宽乘法
  3. 与累加值相加
- flush会清空融合单元的流水状态。
- 融合单元不修改数据总线、DCache或Load/Store接口。

### 软件构建集成

- CoreMark源码保持不变。
- `scripts/fuse_xmac16_asm.py`在CoreMark矩阵源文件编译成汇编后，按数据依赖模式识别并替换基础指令序列。
- 匹配失败时，程序保留原始基础指令，不影响正确性。
- 当前生成的矩阵热点中包含 `macc16` 和 `bfmacc16`，可在 `firmware.dis` 中看到对应custom-0指令。

## 6. 当前实测结果

在50MHz仿真时钟、CoreMark 10轮性能窗口下：

| 版本 | cycles/轮 | 10000轮估算 |
|---|---:|---:|
| 分支/Zbkb基础优化版本 | 约346,744.5 | 约69.35秒 |
| 单周期融合临时版本 | 约303,183.8 | 约60.64秒 |
| 当前三级流水融合版本 | 约304,733.8 | 约60.95秒 |

当前三级流水版本相对基础优化版本：

- 减少约42,010.7 cycles/轮
- 性能提升约12.12%
- 10000轮节省约8.40秒
- CoreMark CRC检查通过
- 裸机冒烟测试通过

三级流水相比单周期组合版本只增加约0.31秒/10000轮，但显著缩短了融合单元的组合路径。最终50MHz是否满足，仍需要Vivado综合、布局布线和时序报告确认。

## 7. 当前固件镜像

当前综合输入镜像为：

```text
build/images/rtthread-coremark/code.mem
build/images/rtthread-coremark/data.mem
build/images/rtthread-coremark/image.json
```

对应软件文件为：

```text
build/software/rtthread-coremark/firmware.elf
build/software/rtthread-coremark/firmware.bin
build/software/rtthread-coremark/firmware.dis
```

`image.json`记录了ELF、代码区和数据区的SHA-256，`scripts/check_images.py`当前可以验证通过。

