# Prompt For The Next Agent

请完整读取本目录中的以下文件：

1. `README.md`
2. `technical_handoff.md`
3. `referenced_files/source_manifest.md`
4. `referenced_files/my_final_dev_2_vs_dev-v3.2-release.diff`
5. `referenced_files/baseline_rtthread_coremark_command_10.json`
6. 在需要追溯时读取 `conversation_transcript.md`

接手后先完成以下工作，不要立即改代码：

1. 复述你对 SocRV 当前分支、旧三条自定义指令、CoreMark 调用链和编译流程的理解。
2. 分开列出“仓库或日志已经证明的事实”“用户报告的数据”“尚未实测的估算”。
3. 指出文档中不同阶段性能估算的变化，并采用 `technical_handoff.md` 中的最新收敛值。
4. 检查实际仓库当前分支和工作树，不要假定它仍停留在 `my_final_dev_2`。
5. 在用户授权修改前，只做只读核查。

如果继续设计通用扩展，默认目标为：

```text
标准 Zba + Zbb
+ 通用 xmacc/xmacc16
+ 通用 xbfxu
```

可选补充按优先级评估 `Zicond`、`Zbs`、`Zbc`。任何收益预测必须给出：动态命中次数、每次节省的指令/周期、编译器生成条件、旁路延迟、主频影响和回退方案。

