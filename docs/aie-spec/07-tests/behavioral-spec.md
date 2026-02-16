# Behavioral Specification Extracted from Tests

This section treats tests as the operational specification of backend behavior.

## 1. Core invariants

## 1.1 Triple-specific behavior must be stable

Evidence:
- many tests run identical inputs for `aie`, `aie2`, `aie2p` with prefixed checks.

Requirement:
- same source pattern may lower to different opcodes/CSR/frame ops by triple,
- these differences are intentional and must remain deterministic.

## 1.2 VLIW bundle legality is a correctness condition

Evidence:
- `MC/AIE/invalid-bundles.s` expects `incorrect bundle`.
- scheduling/resource tests assert exact bundle or non-bundle outcomes.

Requirement:
- illegal slot combinations must be rejected or separated by scheduler/NOP insertion,
- parser, scheduler, and emitter must agree on legal bundle forms.

## 1.3 Relocation encoding must match architecture-specific ABI

Evidence:
- `relocations/elf-relocs.ll`, `vliw_relocfixups.mir`, `data2data.mir`.

Requirement:
- fixup-to-relocation mappings (including bundled offsets) must produce exact relocation kinds and offsets in object files.

## 1.4 Delay-slot and barrier semantics are explicit

Evidence:
- many schedule/prologue tests check ordered `NOP` sequences and `DelayedSchedBarrier` placement.

Requirement:
- branch/call/return delay slot behavior is part of ABI-visible code shape and cannot be optimized away incorrectly.

## 1.5 Machine verifier constraints on tied physical registers

Evidence:
- `aie2*/verifier/tied-physical-regs-match-*.mir`.

Requirement:
- tied phys-reg tuples for 2D/3D and pseudo memory ops must match expected architectural coupling;
- mismatches must be diagnosed.

## 1.6 Register-allocation failure modes are specified

Evidence:
- `spill-sreg-noRreg.mir` expects deterministic RA failure text when no legal spill path exists.

Requirement:
- allocator must fail predictably (and not silently miscompile) when register-class constraints make code impossible.

## 2. Expected optimization/pass behavior

## 2.1 GlobalISel legalization/selection

Expected:
- generic ops lower to arch-specific selected instructions (`inst-select-*.mir`).
- library call lowering for unsupported bulk ops (for example `G_MEMSET` -> call sequence with `PseudoJL &memset`).
- currently unsupported operations are explicitly tracked by fail-intent tests (`xfail-legalize-*`, `xfail-irtranslator-*`).

## 2.2 If-conversion / CFG simplification heuristics

Expected:
- `early-ifcvt` on AIE2P converts eligible switch/phi patterns into select chains under critical-path limit.
- `SimplifyCFG` threshold tuning is architecture-sensitive; at higher threshold, phi-merge forms are expected to convert to speculative-select form.

## 2.3 Hardware-loop conversion

Expected:
- profitable loops produce hardware-loop intrinsics (`llvm.start/ set.loop.iterations`, `llvm.loop.decrement*`).
- unprofitable/unsupported loops remain in canonical branch form and emit analysis remarks where applicable.
- metadata (for example `llvm.loop.itercount.range`) gates ZOL acceptance in some tests.

## 2.4 Scheduling and post-pipelining

Expected:
- negative-latency scenarios force specific spacing/reordering (RAW/WAR/WAW suites).
- inter-block scoreboard constraints affect predecessor/successor placement.
- post-pipeliner may reshape loop prologue/kernel/epilogue and is validated end-to-end.

## 2.5 Resource/bank-aware multi-slot assignment

Expected:
- multi-slot memory ops from conflicting banks/ports are serialized,
- non-conflicting bank combinations are co-issued in one bundle.

## 3. Code pattern recognition requirements

## 3.1 Addressing modes and composite forms

From MC and CodeGen tests:
- parser must accept rich forms:
  - `[ptr, #imm]`, `[sp, #-imm]`, `[ptr], mX`, `[ptr], dX`, 2D/3D variants,
- invalid forms must produce precise diagnostics.

## 3.2 Constant materialization boundaries

From instruction-select tests:
- immediate boundary behavior is fixed by target:
  - direct add-immediate forms for in-range constants,
  - move/materialize + add fallback for out-of-range constants.

## 3.3 Call lowering and ABI class handling

From ABI/prologue tests:
- narrow integer arguments still consume full ABI slots/registers.
- mixed-width argument passing and stack placement are verified with concrete offset checks.

## 4. Register allocation expectations

## 4.1 Staged RA must preserve bundle/legality contracts

Evidence:
- staged RA tests under `aie2*/ra/` and `spill/` heavily check spill placement and rewritten instruction forms.

Requirement:
- rewrite/coalesce/split passes cannot produce illegal bundles or violate tied/subregister invariants.

## 4.2 Spill/reload classes are architecture-specific and intentional

Evidence:
- per-class spill tests (`d`, `dj`, `dn`, `m`, fifo, accumulators, vectors).

Requirement:
- each register class must have valid spill policy or deterministic failure behavior where unsupported.

## 5. Scheduling order expectations

## 5.1 PostMISched output is shape-checked, not just semantically checked

Evidence:
- many MIR tests use strict `CHECK-NEXT` sequencing including inserted NOP counts.

Requirement:
- scheduler changes that alter cycle-level shape require intentional test updates and are treated as behavior changes.

## 5.2 Cross-block scheduling invariants

Evidence:
- interblock scoreboard tests deliberately model successor conflicts and require specific placement decisions.

Requirement:
- schedule legality must be evaluated with CFG context, not locally per block.

## 6. Known limitations captured by tests

Hard limitations tracked today:
- AIE1 alignment/disassembler corner cases marked `XFAIL`.
- several GlobalISel operations intentionally unsupported (fail-intent `xfail-*` files).
- transform notes indicate unresolved loop-preexistence detection corner cases in hardware-loop pass tests.

These are part of current expected behavior and should be considered when deriving a reproduction runbook.
