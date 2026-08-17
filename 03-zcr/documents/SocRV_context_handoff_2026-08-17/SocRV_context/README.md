# SocRV / CoreMark Conversation Handoff

本目录用于把 Codex 任务“询问coremark相关”的可见对话和关键技术证据交给另一位使用者或智能体。

## 建议阅读顺序

1. `technical_handoff.md`：先掌握已经确认的事实、估算和未决问题。
2. `referenced_files/source_manifest.md`：确认仓库、分支、源码和构建文件位置。
3. `referenced_files/my_final_dev_2_vs_dev-v3.2-release.diff`：查看旧三条自定义指令的真实实现。
4. `referenced_files/baseline_rtthread_coremark_command_10.json`：查看历史基线性能数据。
5. `conversation_transcript.md`：需要追溯推导过程、C 语言讲解或早期结论时再阅读全文。
6. `handoff_prompt.md`：把其中的提示词交给接手智能体。

## 内容边界

- `conversation_transcript.md` 是通过 Codex 任务历史接口分页导出的可见用户消息、助手答复和可见工作更新。
- 转录不包含系统提示、开发者提示、隐藏推理或工具命令输出。
- 生成交接包时最后一个回合仍在进行，因此转录末尾不包含本次交付完成后的最终答复。
- 百分比性能结论大多是基于动态次数、退休指令和流水线结构的工程估算，不是严格的同条件 A/B 实测。
- 用户报告“旧三条扩展下 10000 轮约 13 秒”；当前包内没有对应原始日志，接手者应把它视为用户提供的测量值。
- 原始仓库位于 `D:\IKnow\FPGA\Vivado\SocRV`，不在本交接目录内。需要修改、仿真或综合时仍需取得仓库。

## 隐私提示

逐字转录会保留对话中出现的本地绝对路径、用户名、附件路径和 Git 元数据。对外发送前请自行检查并删除不希望分享的信息。

## 快速结论

- 旧方案是 `bfmul16 + crc8step + isdigit8`，通过汇编后处理脚本写入 `custom-0` 指令。
- 旧方案 CoreMark 针对性强；`crc8step` 和 `bfmul16` 局部融合粒度大，但对源码/汇编形态变化较敏感。
- 当前讨论的通用替代方案是标准 `Zba + Zbb`，再加通用 `xmacc/xmacc16` 和 `xbfxu`。
- 同频情况下，新组合预计比旧组合略慢；按用户给出的 13 秒，暂估约 `13.2-13.6 s`，中心约 `13.3 s`。
- 最值得继续评估的通用补充是 `Zicond`、`Zbs`，以及在有编译器 CRC 识别时使用 `Zbc`。
- 所有最终决策必须同时比较 `cycles/iteration` 和布局布线后的可实现频率，不能只比较静态指令数。

