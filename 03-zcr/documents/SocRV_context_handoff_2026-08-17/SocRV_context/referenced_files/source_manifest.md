# Source And Evidence Manifest

## Repository state at handoff

```text
Repository: D:\IKnow\FPGA\Vivado\SocRV
Current branch: final_dev_2
Current HEAD: 001c93ac5246673e5c5781950c67914e46c96316
Tracks: origin/release-v1.0

Old custom-extension branch: my_final_dev_2
Old custom-extension commit: f36187cd164870758e7e233bfa1a2112b26bacd7
Comparison baseline: origin/dev-v3.2-release
Comparison baseline commit: 3a2a51fbf261908a77cbfbfcbae45e4cce5418ce
```

The handoff was created while the repository was clean according to `git status --short --branch`.

## CoreMark core files used by rtthread-coremark

```text
software/coremark/upstream/core_main.c
software/coremark/upstream/core_list_join.c
software/coremark/upstream/core_matrix.c
software/coremark/upstream/core_state.c
software/coremark/upstream/core_util.c
software/coremark/upstream/coremark.h
```

These six files were verified in the conversation as byte-identical to official CoreMark v1.01 at short commit `cfa9ab3`. They are not byte-identical to the current upstream `main`, which contains later formatting and maintenance edits.

## CoreMark port and RT-Thread entry files

```text
software/coremark/port/common/core_portme.c
software/coremark/port/common/ee_printf.c
software/coremark/port/common/cvt.c
software/coremark/port/rtthread/command.c
software/applications/rtthread/shell_main.c
```

The complete firmware also builds RT-Thread, BSP, startup, UART, timer and runtime sources. Those support the system but are not CoreMark algorithm files.

## Old custom instruction implementation

Use the saved diff in this directory or inspect these files at commit `my_final_dev_2`:

```text
rtl/core/eh1f/dec/dec_decode_ctl.sv
rtl/core/eh1f/exu/exu_alu_ctl.sv
rtl/core/eh1f/exu/exu_mul_ctl.sv
rtl/core/eh1f/include/veer_types.sv
scripts/fuse_coremark_asm.py
software/Makefile
software/toolchain/common_flags.mk
```

Useful commands:

```powershell
git show my_final_dev_2:scripts/fuse_coremark_asm.py
git diff origin/dev-v3.2-release...my_final_dev_2 -- scripts software rtl/core/eh1f
```

## Generated assembly evidence

The local repository contains a separate historical `xmac16` experiment:

```text
build/software/rtthread-coremark-xmac16/objects/coremark/upstream/core_matrix.c.o.base.s
build/software/rtthread-coremark-xmac16/objects/coremark/upstream/core_matrix.c.o.xmac.s
```

These files show real `mul + add` replacements with custom `macc16` instructions. They are feasibility evidence, not a controlled performance comparison with the later EH1F branch.

## Performance evidence

Saved in this package:

```text
baseline_rtthread_coremark_command_10.json
```

Original location:

```text
D:\IKnow\FPGA\Vivado\SocRV\backup\2026-08-06_17-53-28\result\soc\rtthread-coremark-command-10.json
```

This result is a historical baseline, not an A/B measurement of the old or proposed instruction suites.

