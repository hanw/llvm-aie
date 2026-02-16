# AIE CodeGen Test Patterns

This document groups high-value CodeGen tests by behavior they validate, emphasizing non-obvious constraints and edge cases.

## 1. Relocation and ELF semantics

Representative tests:
- `llvm/test/CodeGen/AIE/relocations/elf-relocs.ll`
- `llvm/test/CodeGen/AIE/relocations/vliw_relocfixups.mir`
- `llvm/test/CodeGen/AIE/relocations/data2data.mir`

Input pattern:
- IR globals/functions and MIR instructions using symbol operands (for example `MOV_U20_I20 target-flags(aie-global) @myint`, `JAL @target_function`).

Expected output pattern:
- architecture-specific relocation kinds in object output:
  - AIE1 DM word relocation `R_AIE_72`
  - AIE2 DM word relocation `R_AIE_50`
  - AIE2P DM word relocation `R_AIE_62`
- fixups in bundled contexts map to correct relocated offsets (`FIXUP` and `RELOC` checks side-by-side).

Non-obvious behavior:
- relocation kind depends on both instruction form and bundle slot placement; tests verify translated fixups for composite bundles.

## 2. Scheduling with negative latencies and hazard spacing

Representative tests:
- `llvm/test/CodeGen/AIE/aie2/schedule/negative_latencies/RAW.mir`
- `llvm/test/CodeGen/AIE/aie2p/schedule/negative_latencies/RAW.mir`
- `llvm/test/CodeGen/AIE/aie2/schedule/interblock/scoreboard.mir`
- `llvm/test/CodeGen/AIE/aie2p/schedule/interblock/scoreboard.mir`

Input pattern:
- MIR with intentionally conflicting read/write timing and branch-connected blocks.

Expected output pattern:
- deterministic NOP insertion and instruction reordering to satisfy port/latency constraints.
- preserved delayed-branch barriers and exact post-scheduler instruction order in successor/predecessor blocks.

Non-obvious behavior:
- scheduling validity is checked across basic-block boundaries (inter-block scoreboard), not only within one block.

## 3. Software pipelining and loop materialization

Representative tests:
- `llvm/test/CodeGen/AIE/aie2p/schedule/postpipeliner/end-to-end.ll`
- `llvm/test/CodeGen/AIE/aie2/schedule/postpipeliner/*.mir`
- `llvm/test/CodeGen/AIE/aie2p/schedule/postpipeliner/multiSlotAssignment/*.mir`

Input pattern:
- loops with vector loads/compute and loop metadata/trip-count constraints.

Expected output pattern:
- generated prologue/kernel/epilogue with multi-stage software pipeline,
- explicit bundles in kernel and delay-slot materialization in exits,
- expected number/shape of staged bundles for end-to-end examples.

Non-obvious behavior:
- post-RA pipeliner owns final loop shape, not just instruction order; tests treat this as a correctness property.

## 4. Resource/banking-aware multi-slot bundling

Representative tests:
- `llvm/test/CodeGen/AIE/aie2p/schedule/resource/vld_fifo_multi_slot_memory_banks.mir`

Input pattern:
- pseudo multi-slot VLD/VLDB/VLDA FIFO operations with different addrspaces (bank annotations).

Expected output pattern:
- same-bank operations scheduled apart with NOPs,
- cross-bank operations co-issued in one bundle,
- expected explicit bundle structure with correct implicit defs/uses.

Non-obvious behavior:
- memory-bank distinction directly controls co-issuance legality.

## 5. Register allocation and spill edge behavior

Representative tests:
- `llvm/test/CodeGen/AIE/aie2p/spill/spill-sreg-noRreg.mir`
- `llvm/test/CodeGen/AIE/aie2p/ra/staged-ra-cycle-in-bundle.ll`
- `llvm/test/CodeGen/AIE/aie2/ra/*.mir`, `llvm/test/CodeGen/AIE/aie2p/ra/*.mir`

Input pattern:
- pressure-heavy MIR/IR with constrained GPR availability and complex bundled code.

Expected output pattern:
- either successful staged RA with stable spill/reload placement and stack layout,
- or explicit expected failure (`ran out of registers during register allocation`) in constrained negative tests.

Non-obvious behavior:
- staged RA policy materially changes generated code size/shape; tests compare fine-grained vs coarse-grained effects.

## 6. Machine verifier invariants (tied physical registers)

Representative tests:
- `llvm/test/CodeGen/AIE/aie2/verifier/tied-physical-regs-match-vld-2d.mir`
- `llvm/test/CodeGen/AIE/aie2p/verifier/tied-physical-regs-match-2d-vld-pseudo.mir`
- `llvm/test/CodeGen/AIE/verifier/*.mir`

Input pattern:
- MIR with intentionally mismatched tied phys-reg operands for 2D/3D load/store/pseudo families.

Expected output pattern:
- verifier emits counted diagnostics (`Bad machine code: Tied physical registers must match`) for mismatches,
- virtual-register and `renamable` escape-hatch cases accepted where intended.

Non-obvious behavior:
- these verifier checks encode architectural coupling between ptr/mod/dim register tuples.

## 7. Calling convention, prologue/epilogue, and ABI checks

Representative tests:
- `llvm/test/CodeGen/AIE/abi/ints.ll`
- `llvm/test/CodeGen/AIE/call-prologepilog.ll`

Input pattern:
- IR functions passing mixed-width arguments and calling external/defined/fastcc callees.

Expected output pattern:
- fixed stack growth/restoration sequences,
- callee-save save/restore sets by architecture,
- argument placement behavior for narrow and wide integer types,
- expected delayed-slot NOP placement around calls/returns.

Non-obvious behavior:
- AIE2 and AIE2P use distinct frame adjustment ops/stack granularity while preserving ABI intent.

## 8. GlobalISel legalization/selection and known unsupported ops

Representative tests:
- `llvm/test/CodeGen/AIE/GlobalISel/inst-select-add.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/legalize-memset.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-sadde.mir`

Input pattern:
- generic `G_*` MIR ops and pseudo-call lowering scenarios.

Expected output pattern:
- target-specific selected opcodes by triple (`AIE1`, `AIE2`, `AIE2P` split checks),
- `G_MEMSET` lowered to call sequence (`PseudoJL &memset` with expected CSR),
- known unsupported ops fail intentionally in xfail tests.

Non-obvious behavior:
- tests pin per-arch opcode naming and implicit-def conventions, so opcode family drift is treated as a regression.

## 9. Hardware-loop recognition and metadata gating

Representative tests:
- `llvm/test/CodeGen/AIE/hardware-loops/zol-md-accept-reject.ll`
- `llvm/test/Transforms/HardwareLoops/AIE/profitable.ll`
- `llvm/test/Transforms/HardwareLoops/AIE/unprofitable.ll`

Input pattern:
- loops with/without tripcount metadata and profitability conditions.

Expected output pattern:
- accepted loops emit `llvm.loop.decrement` / `llvm.set.loop.iterations` forms,
- rejected loops avoid loop-intrinsic conversion and may emit analysis remarks.

Non-obvious behavior:
- profitability and metadata thresholds are part of contract, not optimization accidents.

## 10. MC-layer conformance from codegen side

Representative tests:
- `llvm/test/CodeGen/AIE/disassembler.ll`
- `llvm/test/CodeGen/AIE/aie2p/AsmBackend/write_nop_sequence.mir`

Input pattern:
- generated object code from IR/MIR.

Expected output pattern:
- objdump round-trip disassembly matches textual expectations,
- exact nop byte sequences and decoded bundle forms are stable for alignment/fill scenarios.

## XFAIL and known-broken list

Lit `XFAIL` tests:
- `llvm/test/CodeGen/AIE/aie1/pad-align.mir`
- `llvm/test/CodeGen/AIE/aie1/frame4.ll`
- `llvm/test/CodeGen/AIE/aie1/BinaryOutput/JAL.mir`
- `llvm/test/CodeGen/AIE/aie1/Disassembler/disassembler_ambiguity.mir`

Failure-intent `xfail-*` GlobalISel tests (expected to fail currently):
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-irtranslator-formal-arguments-vect.ll`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-irtranslator-ret-array.ll`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-irtranslator-ret-vect.ll`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-fconstant.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-insertvector.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-mulh.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-sadde.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-shufflevector.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-ssube.mir`
- `llvm/test/CodeGen/AIE/GlobalISel/xfail-legalize-storevec.mir`
