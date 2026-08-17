# Technical Handoff

## 1. Objective and current direction

The discussion began by comparing `my_final_dev_2` with `origin/dev-v3.2-release`, then traced the complete RT-Thread/CoreMark call flow and the three old custom instructions. The design question later changed from narrowly optimizing CoreMark to building extensions that remain useful for ordinary workloads and for a competition-provided, slightly modified CoreMark source.

Current proposed replacement suite:

```text
Standard Zba + Zbb
+ generic xmacc/xmacc16
+ generic xbfxu
```

Optional general extensions discussed later:

```text
Zicond
Zbs
Zbc, but only if the compiler can recognize or intentionally lower CRC
```

No implementation of the new suite has been made in this handoff task.

## 2. Repository and branch facts

| Item | Value |
|---|---|
| Repository | `D:\IKnow\FPGA\Vivado\SocRV` |
| Branch checked out when this package was created | `final_dev_2` |
| Current HEAD | `001c93ac5246673e5c5781950c67914e46c96316` |
| Old custom-extension branch | `my_final_dev_2` |
| Old custom-extension commit | `f36187cd164870758e7e233bfa1a2112b26bacd7` |
| Comparison branch | `origin/dev-v3.2-release` |
| Comparison commit | `3a2a51fbf261908a77cbfbfcbae45e4cce5418ce` |

`my_final_dev_2` adds or changes seven files relative to the comparison baseline: four EH1F RTL files, the assembly fusion script, the software Makefile and common compiler flags. The complete diff is saved under `referenced_files/`.

Important: the repository was switched to another branch during the conversation. Always inspect the actual current branch before editing; do not assume the old custom implementation is checked out.

## 3. CoreMark source provenance and build inputs

The six official CoreMark core files used by the `rtthread-coremark` profile are:

```text
core_main.c
core_list_join.c
core_matrix.c
core_state.c
core_util.c
coremark.h
```

They are stored under:

```text
software/coremark/upstream/
```

They were verified as byte-identical to official CoreMark v1.01 at short commit `cfa9ab3`. They are not byte-identical to the present upstream `main`; upstream later changed formatting, licenses, comments and some maintenance details, without materially changing the benchmark algorithm used here.

CoreMark-specific port and command files include:

```text
software/coremark/port/common/core_portme.c
software/coremark/port/common/ee_printf.c
software/coremark/port/common/cvt.c
software/coremark/port/rtthread/command.c
software/applications/rtthread/shell_main.c
```

The complete firmware also builds RT-Thread and BSP/startup/runtime sources.

## 4. Runtime entry and measured interval

CoreMark does not run automatically when RT-Thread boots. The practical call flow is:

```text
RT-Thread boot
  -> applications/rtthread/shell_main.c: main()
  -> user enters: coremark <iterations>
  -> coremark/port/rtthread/command.c: cmd_coremark()
  -> coremark_main()
     (core_main.c main renamed at compile time with -Dmain=coremark_main)
  -> initialize seeds, list, matrix and state data
  -> start_time()
  -> iterate(&results[0])
  -> stop_time()
  -> CRC validation and result printing
  -> return to MSH
```

Nearly all scored time is spent in `iterate()` and callees. Its central loop is:

```c
for (i = 0; i < iterations; i++) {
    crc = core_bench_list(res, 1);
    res->crc = crcu16(crc, res->crc);
    crc = core_bench_list(res, -1);
    res->crc = crcu16(crc, res->crc);
}
```

`core_bench_list(res, 1)` sorts with `cmp_complex`. That callback invokes `calc_func`, which dispatches list data to `core_bench_matrix` or `core_bench_state`. The later `cmp_idx(..., NULL)` restores the original data and clears cached calculation results, so matrix/state work recurs on every benchmark iteration.

## 5. Old custom instructions

All three use the RISC-V `custom-0` major opcode `0x0b` and distinguish operations by `funct3`.

| funct3 | Instruction | Meaning | Main execution path |
|---:|---|---|---|
| 0 | `bfmul16` | 16-bit multiply, two fixed bit-field extracts, then multiply extracted fields | MUL pipeline |
| 1 | `crc8step` | Update CRC16 with one byte using reflected polynomial `0xA001` | MUL pipeline/control and writeback |
| 2 | `isdigit8` | Return 1 when the low byte is ASCII `0` through `9` | ALU |

### 5.1 bfmul16

Source target in `core_matrix.c`:

```c
MATRES tmp = (MATRES)A[i*N+k] * (MATRES)B[k*N+j];
C[i*N+j] += bit_extract(tmp,2,4) * bit_extract(tmp,5,7);
```

Matched assembly shape:

```asm
mul   p, rs1, rs2
srai  x, p, 2
srai  y, p, 5
andi  x, x, 15
andi  y, y, 127
mul   rd, x, y
```

The script replaces those six calculations with one custom instruction. Loads, address generation, accumulation into C, stores and loop control remain.

Hardware implementation in the old branch:

- Reuses the existing MUL issue/control/writeback path.
- Adds `cm_op` pipeline state.
- Adds a separate signed 16 x 16 product in E2.
- In E3, extracts fixed fields from that product and performs the second multiply before selecting the result.
- This is the highest timing/resource-risk old extension because it adds multiplication hardware and E3 combinational work.

A previously proposed hardware improvement is to reuse the low 32 bits of the existing signed multiplier product instead of maintaining `bf_prod_e2/bf_prod_e3`. This should preserve semantics while potentially removing an extra DSP and reducing local routing, but it has not been re-synthesized and verified.

### 5.2 crc8step

Software semantics:

```text
new_crc = CRC16_IBM_reflected_step(old_crc, data_byte, polynomial=0xA001)
```

The original `crcu8()` executes eight bit rounds. The custom instruction performs four rounds in the first CRC logic stage and four rounds in the next, then returns the 16-bit result through the MUL result path.

Hardware implementation:

- Reuses MUL pipeline control, latency tracking and writeback.
- Adds new CRC combinational logic; it does not reuse the multiplier arithmetic.
- E1 processes the low four data bits.
- E2 processes the high four data bits.
- E3 selects the CRC result.

The assembly script replaces whole helper bodies for `crcu8`, `crcu16`, `crc16` and `crcu32`. This is fast but semantically risky: changing the official C implementation does not necessarily make the build fail; the script can still overwrite the function with the old fixed CRC behavior.

### 5.3 isdigit8

Source target in `core_state.c`:

```c
retval = ((c >= '0') & (c <= '9')) ? 1 : 0;
```

Typical compiler form:

```asm
addi  result, character, -48
andi  result, result, 255
li    limit, 9
bgtu  result, limit, not_digit
```

The custom operation produces a Boolean, and the range branch becomes `beqz` or `bnez`. The script intentionally keeps the `li 9` because it does not perform complete cross-basic-block liveness analysis.

Hardware implementation adds low-byte range comparison and result selection in the ALU E1 path. This was considered the lowest hardware risk of the old three instructions.

## 6. Old compiler/toolchain integration

Baseline flags discussed in the repository:

```make
-O3 -march=rv32imf_zicsr -mabi=ilp32f
```

The old branch does not teach GCC a new advertised ISA extension. Instead it:

1. Compiles selected CoreMark C files to assembly with `-S`.
2. Runs `scripts/fuse_coremark_asm.py`.
3. Replaces matched assembly with `.insn r 0x0b, ...`.
4. Assembles the rewritten source.
5. Fails the build when the expected class of candidate is not found.

`core_util.c` is compiled with `-fno-inline -g0` so CRC helper functions remain available for whole-function replacement. The Makefile also adds `EXTRA_OPT_FLAGS` support.

Risks:

- Register allocation, instruction scheduling, loop unrolling or branch layout can make a valid semantic pattern stop matching.
- A missed local pattern usually loses performance but remains correct.
- Whole-function CRC replacement can remain matched while semantics changed, creating a correctness failure.
- Numeric changes to the `bfmul16` shifts or masks cannot be substituted by the fixed hardware instruction.

Preferred long-term integration is a compiler backend pattern, explicit intrinsic, or semantics-checked fallback. A decode-stage macro-op fusion implementation is another toolchain-independent option, but retirement and exception handling become more complex.

## 7. Dynamic workload facts

Configuration used for the refined counts:

```text
Official CoreMark v1.01
performance/validation-style seeds discussed: 0, 0, 0x66
TOTAL_DATA_SIZE = 2000
three algorithms enabled
per-algorithm block approximately 666 bytes
matrix dimension N = 9
```

The list contains 28 ordinary calculation items. Their low three flag bits yield four matrix items and four state-machine items per complete CoreMark iteration.

Therefore each CoreMark iteration performs four `matrix_test(N=9)` calls.

### Matrix operation counts per matrix_test

| Function | Multiply-accumulate opportunities |
|---|---:|
| `matrix_mul_vect` | 81 |
| `matrix_mul_matrix` | 729 |
| `matrix_mul_matrix_bitextract` final accumulation | 729 |
| Total | 1539 |

Across four matrix tests:

```text
generic MAC opportunities = 1539 x 4 = 6156
bit-field extracts = 729 x 2 x 4 = 5832
old bfmul16 expression executions = 729 x 4 = 2916
```

Refined old-instruction savings:

| Old instruction | Dynamic count or equivalent | Approximate retired-instruction reduction |
|---|---:|---:|
| `bfmul16` | 2916 | 14580, about 1.95% of the historical 746596 commits/iteration |
| `isdigit8` | about 3920 matched checks | about 3920, roughly 0.53% |
| `crc8step` | about 588 byte updates | about 34000-38300 depending on baseline inlining/path assumptions, roughly 4.6-5.1% retired instructions |

Cycle improvement is smaller than simply multiplying retired-instruction percentages. Dual issue, branch penalties, dependency latency and memory stalls must be modeled or measured.

## 8. Historical performance evidence

The saved historical JSON reports for a 10-iteration run:

```text
cycles/iteration  = 2,378,669.1
commits/iteration =   746,596.3
IPC               = 0.313871
```

Another repository backup discussed in the conversation reported about 1,707,031.6 cycles/iteration with the same commit count on a different CPU state. These are not interchangeable controlled baselines.

User-provided observation:

```text
old bfmul16 + crc8step + isdigit8
10000 iterations
approximately 13 seconds
```

No raw log for that 13-second observation was found during this handoff. Timer frequency, FPGA frequency, source hash and build hash should be recorded before using it as competition evidence.

## 9. Latest performance estimates

These ranges are engineering estimates, not measured A/B results.

### Old suite, relative to an unextended baseline

| Component | Whole-CoreMark cycle/score estimate |
|---|---:|
| `bfmul16` | about 1.5-3% |
| `crc8step` | about 2-3.5% despite a larger retired-instruction reduction |
| `isdigit8` | small, roughly below 1% |
| Combined old suite | about 4.5-7%, without directly adding upper bounds |

Early parts of the conversation gave `bfmul16` a wider 2-5% estimate. That was later narrowed after counting exactly 2916 dynamic expressions per iteration. Use the refined range above.

### Proposed new suite

| Component | Estimated whole-CoreMark gain |
|---|---:|
| standard `Zba + Zbb` | about 0.5-2% |
| generic `xmacc/xmacc16` | about 1-1.8%, optimistic about 2.5% |
| generic `xbfxu` | about 0.6-1.2%, optimistic about 1.5% |
| combined suite | about 2.5-5%, center near 3.5% |

`xmacc` saves at most 6156 retired instructions per iteration if every matrix candidate is emitted. `xbfxu` saves 5832. Combined static/dynamic saving is about 11988 instructions, or 1.61% of the historical 746596 commits/iteration, before Zba/Zbb and cycle-latency effects.

The old specialized `bfmul16` saves about 14580 matrix instructions. New `xmacc + xbfxu` saves about 2592 fewer matrix instructions, but works in ordinary matrix MAC loops and has much broader semantics.

### Conversion of the user-reported 13 seconds

At unchanged clock and assuming old-suite and new-suite center estimates:

```text
old suite gain ~= 5.5%
new suite gain ~= 3.5%
implied unextended time ~= 13 x 1.055 = 13.72 s
new time ~= 13.72 / 1.035 = 13.25 s
```

Practical estimate:

```text
likely:      13.2-13.6 s
center:      about 13.3 s
poor match:  13.5-13.8 s
```

A 5% frequency loss would push approximately 13.3 seconds to about 14.0 seconds. A 5% frequency improvement after removing old critical logic could reduce it to about 12.7 seconds. Final score is proportional to achievable frequency divided by cycles/iteration.

## 10. Proposed generic instruction semantics and caveats

### Standard Zba + Zbb

Potentially useful operations include `sh1add/sh2add/sh3add`, `sext.b/sext.h`, `zext.h`, `min/max`, `andn/orn/xnor`, rotations and bit counts.

Advantages:

- Standard ISA.
- GCC can generate many patterns directly.
- Broad usefulness outside CoreMark.

Caveats:

- Enabling full Zbb requires implementing all instructions the selected compiler may emit, not just a convenient subset.
- `-O3` already converts many matrix indexes into pointer increments, limiting Zba gain.
- The exact GCC version and `-march` parser/support must be verified.

### Generic xmacc/xmacc16

Conceptual semantics:

```text
xmacc rd, rs1, rs2   : rd = old(rd) + rs1 * rs2
xmacc16              : same idea with defined signed/unsigned 16-bit operands
```

Main hardware challenge is the logical third source operand `old(rd)`, not multiplication itself. It affects register reads, scoreboard dependencies and bypasses. Prefer reusing the existing MUL product and adding the accumulator in a later stage. Adding a separate multiplier can cost DSPs and routing.

### Generic xbfxu

Conceptual semantics:

```text
xbfxu rd, rs1, lsb, width
rd = (rs1 >> lsb) & ((1 << width) - 1)
```

Unlike old `bfmul16`, changing extraction positions should only change instruction immediates, not hardware semantics. The exact encoding and signed/unsigned definition remain to be designed.

## 11. Other generic options discussed

| Option | Expected incremental CoreMark value | Main caveat |
|---|---:|---|
| `Zicond` | about 0.5-1.5% | Current GCC 12.2 may need upgrade/backport to emit it well |
| `Zbs` | about 0.2-0.7% | Small gain, overlaps some bit extraction work |
| `Zbc` | near 0 without software/compiler recognition; potentially 1.5-3% with CRC lowering | GCC does not normally transform the present bit loop into CLMUL automatically |
| zero-overhead hardware loop | about 0.5-1.5% | Loop state, interrupts, flushes and compiler support |
| post-increment load | about 0.8-2% | Two architectural destinations, LSU/writeback/precise exception complexity |
| packed `xdotp16` | about 0.3-1.2% | Packing and memory layout limit CoreMark auto-use; overlaps xmacc16 |
| decode-stage macro-op fusion | workload-dependent | No custom ELF required, but decode/retire/exception logic is harder |

Recommended low-risk expansion beyond the current new suite:

```text
Zba + Zbb + Zbs + Zicond
+ xmacc16
+ xbfxu
```

`Zbc` is attractive only when accompanied by a legitimate compiler/library plan and benchmark-rule review.

## 12. Competition-source robustness

The current repository competition interface was reported to accept a replacement `core_main.c`, while the algorithm files remained the repository's official v1.01 copies. Verify the actual event rules and input mechanism before relying on this.

For any on-site source replacement:

1. Record hashes of all six CoreMark core files.
2. Compile a standard fallback binary first.
3. Check semantic signatures and report static fusion counts.
4. If a custom pattern is absent or semantics differ, fall back to standard code rather than forcing a substitution.
5. Run both validation and performance seeds and verify official CRCs.
6. Treat build success as insufficient; CRC equality and runtime output must also pass.

Whole-function CRC replacement is the most important correctness risk. If retained, an exhaustive or formally equivalent software/hardware check should gate its use.

## 13. Required next measurements

To make a defensible architecture decision:

1. Establish one immutable baseline: same RTL commit, CoreMark source hash, compiler, flags, memory image, board clock and constraints.
2. Measure baseline `cycles/iteration`, `minstret`, IPC and final FPGA frequency.
3. Add one extension at a time and repeat software CRC tests and RTL simulation.
4. Count dynamic candidate PCs for MAC, bit-field extraction, Zba/Zbb and CRC/state operations.
5. Measure dependency latency and stalls; do not convert instruction savings using only average IPC.
6. Run synthesis and implementation with identical constraints for every variant.
7. Compare final throughput as `Fmax / cycles_per_iteration`.
8. Save utilization, WNS/TNS, power if available, bitstream hash, ELF hash and full CoreMark output.

Until those measurements exist, performance percentages in this package should be presented as design estimates.

