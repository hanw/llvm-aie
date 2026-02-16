# AIE Test Suite Overview

Scope searched:
- `llvm/test/CodeGen/AIE/**`
- `llvm/test/MC/AIE/**`
- `llvm/test/CodeGen/MIR/AIE/**`
- `llvm/test/Object/AIE/**`
- `llvm/test/Transforms/HardwareLoops/AIE/**`
- `llvm/test/Transforms/SimplifyCFG/AIE/**`
- `llvm/test/tools/llvm-dwarfdump/AIE/**`
- plus global `*aie*` / `*AIE*` filename scan under `llvm/test`

## 1. Directory layout

Primary AIE backend tests live under:
- `llvm/test/CodeGen/AIE/`
- `llvm/test/MC/AIE/`

Supporting AIE-focused tests also exist under:
- `llvm/test/CodeGen/MIR/AIE/`
- `llvm/test/Object/AIE/`
- `llvm/test/Transforms/HardwareLoops/AIE/`
- `llvm/test/Transforms/SimplifyCFG/AIE/`
- `llvm/test/tools/llvm-dwarfdump/AIE/`

`CodeGen/AIE` sub-layout is feature-oriented:
- architecture partitions: `aie1/`, `aie2/`, `aie2p/`
- pipeline/pass partitions: `GlobalISel/`, `schedule/`, `ra/`, `spill/`, `verifier/`, `relocations/`, `postrapseudos/`, `hardware-loops/`, `opt/`, `bundling/`, `abi/`, `vect/`, `float/`

Notable deep hierarchies:
- `aie2/schedule/{negative_latencies,interblock,postpipeliner,resource,pre_ra,...}`
- `aie2p/schedule/{negative_latencies,interblock,postpipeliner,resource,pre_ra}`
- `aie2p/GlobalIsel/global-combiners/`

## 2. Test categories and counts

### Core AIE test buckets

- `CodeGen/AIE`: **1828** files
- `CodeGen/MIR/AIE`: **3** files
- `MC/AIE`: **13** files
- `Object/AIE`: **3** files
- `Transforms/HardwareLoops/AIE`: **4** files
- `Transforms/SimplifyCFG/AIE`: **2** files
- `tools/llvm-dwarfdump/AIE`: **2** files

Total across these AIE-focused directories: **1855** files.

### Broader filename match

Global `*aie*` filename search under `llvm/test` returns **1866** files.
The extra files are mostly adjacent coverage (for example AIE-related TableGen/analysis/transform tests outside the core directories above).

### CodeGen/AIE distribution

Top-level `CodeGen/AIE` subtree file counts:
- `aie2`: 821
- `aie2p`: 541
- `GlobalISel` (top-level, shared): 271
- `aie1`: 76
- `vect`: 21
- `schedule` (top-level, shared): 18
- `relocations`: 15
- others smaller (`abi`, `opt`, `spill`, `verifier`, `postrapseudos`, `bundling`, `float`, `mir`, `ra`)

By file type under `CodeGen/AIE`:
- `.mir`: 1391
- `.ll`: 432
- `.cfg`: 4

This is primarily a MIR pass/regression suite with targeted IR-to-assembly checks.

## 3. Testing methodology

AIE tests follow standard LLVM `lit` + `FileCheck` style with heavy pass isolation.

Common run styles:
- End-to-end codegen:
  - `llc -mtriple=aie|aie2|aie2p ... | FileCheck`
- Pass-isolated MIR tests:
  - `llc -run-pass=<pass> ... | FileCheck`
  - frequent passes: `instruction-select`, `legalizer`, `postmisched`, `machineverifier`, `prologepilog`, `early-ifcvt`
- MC assembler/disassembler tests:
  - `llvm-mc ...` + `llvm-objdump -dr ...`
- Object/relocation tests:
  - `llvm-readobj -r`, `llvm-readelf -r`, `llvm-objdump -dr`
- Transform tests:
  - `opt -passes=... | FileCheck`
- negative/error-path tests:
  - `not llvm-mc ...`
  - `not --crash llc ...`

Pattern style:
- assembly/MIR structural assertions (`CHECK`, `CHECK-NEXT`, `CHECK-LABEL`, `CHECK-NOT`, `CHECK-COUNT`)
- multi-target comparison in one file (`AIE1`, `AIE2`, `AIE2P` prefixes)
- many MIR checks auto-refreshed (`update_mir_test_checks.py`, `update_llc_test_checks.py` comments present)

## 4. Expected-failure coverage

Lit `XFAIL` markers in core AIE directories: **4** files
- `llvm/test/CodeGen/AIE/aie1/pad-align.mir`
- `llvm/test/CodeGen/AIE/aie1/frame4.ll`
- `llvm/test/CodeGen/AIE/aie1/BinaryOutput/JAL.mir`
- `llvm/test/CodeGen/AIE/aie1/Disassembler/disassembler_ambiguity.mir`

Additionally, there are explicit failure-intent files named `xfail-*` in GlobalISel (10 files), typically asserting current legalization/IRTranslator gaps via `not --crash llc ...`.
