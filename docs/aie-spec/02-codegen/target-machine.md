# AIE Target Machine Configuration

## Scope
This document summarizes target machine configuration for the AIE backend, primarily from:
- `llvm/lib/Target/AIE/AIEBaseTargetMachine.*`
- `llvm/lib/Target/AIE/AIE2TargetMachine.*`
- `llvm/lib/Target/AIE/aie1/AIE1TargetMachine.*`
- `llvm/lib/Target/AIE/aie2p/AIE2PTargetMachine.*`
- Subtarget files:
  - `llvm/lib/Target/AIE/AIEBaseSubtarget.*`
  - `llvm/lib/Target/AIE/AIE2Subtarget.*`
  - `llvm/lib/Target/AIE/aie1/AIE1Subtarget.*`
  - `llvm/lib/Target/AIE/aie2p/AIE2PSubtarget.*`
- API boundary wiring in:
  - `llvm/lib/Target/AIE/AIE.h`
  - `llvm/lib/Target/AIE/AIE2.h`

## Public Interface (Key Overrides)
`AIEBaseTargetMachine`:
- `getObjFileLowering()`
- `createDefaultFuncInfoYAML()`
- `convertFuncInfoToYAML()`
- `parseMachineFunctionInfo()`
- `createMachineFunctionInfo()`
- `registerDefaultAliasAnalyses()`
- `registerPassBuilderCallbacks()`
- `isNoopAddrSpaceCast()`
- `setMBBPlacementOpts()`

`AIEBasePassConfig`:
- `addIRTranslator()`
- `addLegalizeMachineIR()`
- `addRegBankSelect()`
- `addGlobalInstructionSelect()`
- `addIRPasses()`
- `addMachineLateOptimization()`
- `addInstSelector()`
- `addPreEmitPass()`
- `addPreEmitPass2()`
- `addPreRegAlloc()`
- `addPreSched2()`
- `createPostMachineScheduler()`
- `createMachineScheduler()`
- `getCSEConfig()`

`AIE2TargetMachine`:
- `getSubtargetImpl()`
- `getTargetTransformInfo()`
- `createPassConfig()`
- `targetSchedulesPostRAScheduling()`
- `getAddressSpaceForPseudoSourceKind()`

`AIE2PassConfig`:
- `addPreISel()`
- `addPreEmitPass()`
- `addGlobalInstructionSelect()`
- `addPreRegAlloc()`
- `addRegAssignAndRewriteOptimized()`
- `addPostRewrite()`
- `addMachineLateOptimization()`
- `addPreSched2()`
- `addBlockPlacement()`
- `addPreLegalizeMachineIR()`
- `addPreRegBankSelect()`
- `addISelPrepare()`
- `addILPOpts()`

`AIE2PTargetMachine` and `AIE2PPassConfig`:
- `createPassConfig()`
- `getTargetTransformInfo()`
- `addPreLegalizeMachineIR()`
- `addPreRegBankSelect()`
- `addRegAssignAndRewriteOptimized()`

`AIETargetMachine` / legacy `AIEPassConfig` (`aie1`):
- `createPassConfig()`
- `addInstSelector()` (SelectionDAG path via `createAIEISelDag`)
- `addPreEmitPass()` (delay-slot filler + AIE1 block placement)

## Data Layout String

```
e-m:e-p:20:32-i1:8:32-i8:8:32-i16:16:32-i32:32:32-f32:32:32-i64:32-f64:32-a:0:32-n32
```

Breakdown:

| Component | Meaning |
|-----------|---------|
| `e` | Little-endian |
| `m:e` | ELF mangling |
| `p:20:32` | Pointers are 20 bits wide, 32-bit aligned (stored in 32-bit slots) |
| `i1:8:32` | `i1` stored in 8 bits, 32-bit aligned |
| `i8:8:32` | `i8` stored in 8 bits, 32-bit aligned |
| `i16:16:32` | `i16` stored in 16 bits, 32-bit aligned |
| `i32:32:32` | `i32` native width and alignment |
| `f32:32:32` | `f32` native width and alignment |
| `i64:32` | `i64` only 32-bit aligned (split across two 32-bit regs in CC) |
| `f64:32` | `f64` only 32-bit aligned |
| `a:0:32` | Aggregates have 32-bit preferred alignment |
| `n32` | Native integer width is 32 bits |

The 20-bit pointer width is a key architectural feature — address pointers (`eP`) are physically 20 bits wide, stored in 32-bit register slots. This is set identically across all AIE variants via `computeDataLayout()`.

## Relocation Model and Code Model
- `getEffectiveRelocModel()` forces static relocation.
- If PIC is requested, it is downgraded to static (PIC is unsupported).
- Code model defaults via `getEffectiveCodeModel(CM, CodeModel::Small)`.

## New-PM PassBuilder Callbacks
`AIEBaseTargetMachine::registerPassBuilderCallbacks()` wires AIE behavior into the
new pass manager IR pipeline:
- Alias analysis registration:
  - Registers `AIEBaseAA` with the function analysis manager.
  - Supports textual AA pipeline name `aie-aa` via parse callback.
- Optional internalization flow (`-aie-internalize-symbols`):
  - Adds `InternalizePass` + `GlobalDCEPass` in early simplification.
  - Uses skip-list defaults (for example, preserving `main`).
- Optional extra IPSCCP run (`-aie-enable-ipsccp`):
  - Adds `IPSCCPPass` in optimizer-early extension point.

Design intent:
- Keep target-specific alias precision and whole-module dead-stripping close to
  IR optimization, before backend-specific MIR scheduling/packing.

## Command-Line Options

The backend exposes 30+ command-line options for controlling optimization behavior. Key categories:

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-internalize-symbols` | off | Enable symbol internalization for whole-module optimization |
| `-aie-enable-ipsccp` | off | Extra interprocedural constant propagation pass |
| `-aie-address-chaining` | on (O>0) | Enable `AIEClusterBaseAddress` for post-increment formation |
| `-aie-global-ptr-mod-opt` | on (O>0) | Enable `AIEPtrModOptimizer` analysis |
| `-aie-stack-minimize` | on | Enable spill-slot optimization |
| `-aie-stack-addrspace` | 0 | Address space for stack pseudo sources |
| `-aie-enable-pipeliner` | on (O2+) | Enable software pipelining (MachinePipeliner) |
| `-aie-enable-waw-rewriter` | varies | Enable WAW register conflict resolution |

## Subtarget Features and Feature Strings
- `AIETargetMachine` (`aie1` path) builds subtarget with hardcoded `"aie"` CPU fallback in constructor.
- `AIE2TargetMachine` constructs subtarget with CPU default `aie2`.
- `AIE2PTargetMachine` constructs subtarget with CPU default `aie2p`.
- Subtarget constructors call generated `ParseSubtargetFeatures(CPUName, CPUName, FS)`.
- GlobalISel components are created in subtarget constructors:
  - `AIECallLowering`
  - `AIE2/AIE2P LegalizerInfo`
  - `AIE2/AIE2P RegisterBankInfo`
  - `createAIE2/createAIE2PInstructionSelector`

### Subtarget Base Class (`AIEBaseSubtarget`)

Abstract base providing:
- Target query: `isAIE1()`, `isAIE2()`, `isAIE2P()`
- Scheduling policy: `overrideSchedPolicyBase()` — adjusts generic scheduler heuristics for VLIW
- Dependency adjustment: `adjustSchedDependency()` — modifies latencies for target-specific cases (e.g., semaphore ordering edges)
- DAG mutation factories:
  - `getPreRAMutations()` — pre-RA scheduling mutations
  - `getPostRAMutations()` — post-RA scheduling mutations
  - `getSMSMutations()` — software pipelining (SMS) mutations

## Pass Pipeline (AIE2/AIE2P)

### IR and Pre-ISel
1. Atomic expansion (always).
2. Optional AIE alias analysis wrappers at opt > O0.
3. `InferAddressSpaces` at opt > O0.
4. Target IR pipeline.
5. Optional `LoadStoreVectorizer` (non-AIE1, opt > O0).
6. `AIE2`: hardware loop legacy pass in `addPreISel()` at opt > O0.

### GlobalISel Core
1. `IRTranslator`
2. `Legalizer`
3. `RegBankSelect`
4. `InstructionSelect`

### Pre-Legalize MIR (`addPreLegalizeMachineIR`)
1. `AIEAddressSpaceFlattening`
2. `AIE2/AIE2PPreLegalizerCombiner` (opt > O0)
3. `AIEEliminateDuplicatePHI`

### Pre-RegBankSelect (`addPreRegBankSelect`)
1. `AIE2/AIE2PPostLegalizerGenericCombiner`
2. Optional `AIEClusterBaseAddress` (`-aie-address-chaining`)
3. Optional `AIEPtrModOptimizer` (`-aie-global-ptr-mod-opt`)
4. `AIE2/AIE2PPostLegalizerCustomCombiner`

### Global Instruction Selection (`addGlobalInstructionSelect`)
1. `InstructionSelect`
2. DCE (keep lifetime intrinsics)
3. `AIEPostSelectOptimize`
4. DCE again
5. Optional insertion of `ReservedRegsLICMPass`

### Pre-RA and RA
1. Optional machine pipeliner + DCE at `-O2+`.
2. Insert `AIESubRegConstrainer` before `PHIElimination`.
3. Optional `AIEOutlineMemoryGEP` in `addISelPrepare()`.
4. Optional `EarlyIfConverter` via `addILPOpts()`.
5. Staged RA flow (when enabled):
   - Optional coalescer rerun.
   - Optional split-instr builder.
   - Regclass constrainer (AIE2 flow).
   - Selective greedy allocation phases by register family (M, 3D, 3D+2D).
   - Super-reg rewriter between phases.
   - Optional WAW reg rewriter + extra RA.
   - Final virtual register rewriter.
   - AIE2P adds optional unallocated-superreg rewriter in fine-grained mode.

### Post-RA and Pre-Emit
1. Optional spill-slot optimization (`-aie-stack-minimize`).
2. Optional split-instr replacer.
3. DCE.
4. Optional block placement (non-O0).
5. `AIEBaseHardwareLoopsPass` (non-O0).
6. `AIEPseudoBranchExpansion`.
7. Post machine scheduler (always — **mandatory for correctness**).
8. `AIEFinalizeBundle`.
9. `AIEMachineAlignment`.

### Legacy `aie1` Path
1. Uses SelectionDAG instruction selector (`createAIEISelDag`) through `addInstSelector()`.
2. Uses pre-emit delay-slot filler (`createAIEDelaySlotFillerPass`) and AIE1 block-placement pass.
3. Does not use the modern AIE2/AIE2P GlobalISel-centric pipeline.

## Address Space Handling

### Address Space Definitions (`AIE2AddrSpace.h`)

| Address Space | Name | Description |
|---------------|------|-------------|
| 0 | `none` | Default (flat) address space |
| 1 | `PM` | Program memory |
| 2 | `DM` | Data memory |
| 3 | `stack` | Stack memory |
| 4–7 | `a`, `b`, `c`, `d` | Individual memory banks |
| 8–13 | `ab`, `ac`, `ad`, `bc`, `bd`, `cd` | Bank pair combinations |
| 14 | `TM` | Tile memory |

Each address space maps to a bitmask of `AIEBanks {A, B, C, D, TileMemory}`.

### Runtime Behavior
- `isNoopAddrSpaceCast()` always returns true.
- Address spaces carry banking annotations; the `AIEAddressSpaceFlattening` pass rewrites all to default address space 0 before codegen.
- `AIE2TargetMachine::getAddressSpaceForPseudoSourceKind()`:
  - Stack/fixed-stack pseudo values map to configurable `aie-stack-addrspace`.
  - Tile memory pseudo values map to AIE2 TM address space.
  - Others map to `none`.

## Key Design Decisions
- GlobalISel-first design for modern targets (`aie2`, `aie2p`); no fallback SelectionDAG path in pass config.
- Legacy `aie1` remains a separate SelectionDAG-based implementation path, preserving older target behavior without constraining modern pipeline design.
- Strict static relocation model simplifies ABI/link assumptions.
- Two-stage custom combining around legalization plus pointer-mod analysis reflects strong dependence on address-generation quality.
- Post-RA scheduler is mandatory for correctness (NOP insertion and final bundle legality) — this is an exposed pipeline architecture.
- Staged RA is not cosmetic: it explicitly decomposes allocation by register families and rewrites super-regs to control pressure across the deeply partitioned register file.

## Invariants Downstream Code Depends On
- Instruction stream entering packet/finalize must pass through post-RA scheduling.
- Address-space flattening runs before major MIR combining.
- For optimized pipelines, post-legalizer custom combiner may consume `AIEPtrModOptimizer` analysis results.
- Split-instr builder and replacer must be paired when enabled.
- CSE is restricted at `-O0` to avoid constrained-register patterns that `RegAllocFast` cannot resolve.
- AIE2/AIE2P pass wiring depends on pass-factory entry points declared in `AIE.h` / `AIE2.h` remaining stable.
- 20-bit pointer width is assumed throughout the backend; data layout string is computed once and shared across all subtargets.
