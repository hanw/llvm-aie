# AMD/Xilinx llvm-aie Backend Architecture (Unified Synthesis)

This document synthesizes the layered analyses under `docs/aie-spec` into one architecture view for reproducing the AIE backend.

Inputs used:
- `docs/aie-spec/00-overview/manifest.json`
- `docs/aie-spec/01-isa/*.md`
- `docs/aie-spec/02-codegen/*.md`
- `docs/aie-spec/03-scheduling/*.md`
- `docs/aie-spec/05-mc-layer/*.md`
- `docs/aie-spec/06-clang/*.md`
- `docs/aie-spec/07-tests/*.md`
- `docs/aie-spec/08-build/build-system.md`

## 1. Executive Summary

The AIE backend is a specialized LLVM/Clang target stack for AMD/Xilinx AI Engine processors (`aie2`, `aie2p`, with legacy `aie/aie1` support). It exists because AI Engine constraints are not fully captured by generic LLVM target machinery:

- **Slot-typed VLIW packet legality** — instructions belong to slot classes (lda, ldb, alu, mv, st, vec, lng, nop) composed into variable-width packets (16–128 bits). Legality is a data-driven slot-set-to-format lookup, not ad hoc pairwise rules.

- **Exposed pipeline with signed latencies** — register accesses at different pipeline stages create operand-to-operand latencies ranging from +37 down to -8 cycles. Upstream LLVM assumes non-negative latencies, which over-constrains scheduling. AIE uses signed arithmetic throughout readiness computation.

- **Deep, partitioned register file** — AIE2 has 47+ register classes with tuple-coupled operand constraints; AIE2P adds ACC2048, VEC288/576/1152, FIFO, and EXPVEC classes (60+ base classes). Operand register class drives scheduling: AIE2P has 4,010 itinerary classes vs AIE2's 278.

- **Post-RA scheduling as correctness** — the hardware does not interlock on most hazards. The compiler must produce a conflict-free schedule; invalid scheduling produces illegal code. Bundle legality, hazard recognition, and NOP insertion are enforced late.

- **20-bit pointer architecture** — pointers are 20 bits stored in 32-bit register slots (`p:20:32` in data layout). Address manipulation builtins truncate/extend between i32 and i20 at the frontend.

- **Architecture-specific MC encoding** — variable-width composite packet encoding, nibble-based size dispatch, and custom fixup-to-relocation mapping for field-level instruction patching.

### Scale

| Metric | Value |
|--------|-------|
| Source files | 318 |
| Estimated LOC | ~134K |
| Clang builtins | ~655 (AIE1: ~41, AIE2: ~395, AIE2P: ~219) |
| AIE2 itinerary classes | 278 |
| AIE2P itinerary classes | 4,010 |
| AIE2 functional units | 26 |
| AIE2P functional units | 73 |
| AIE2 register classes | 47+ base + 36 operand-constrained variants |
| AIE2P register classes | 60+ base (adds ACC2048, VEC288/576/1152, FIFO, EXPVEC) |
| AIE2 composite formats | 58 total (128-bit: 2, 112: 8, 96: 20, 80: 18, 64: 12, 48: 9, 32: 6, 16: 1) |
| VLIW packet sizes | 2/4/6/8/10/12/14/16 bytes |
| Test suite | 1,855 files (1,391 MIR + 432 LLVM IR + 4 CFG) |

## 2. Compilation Flow

```text
C/C++ Source
  │
  v
Clang Frontend (AIETargetInfo + AIEToolChain)
  - triples: aie/aie2/aie2p (*-none-unknown-elf)
  - predefined macros: __aie__, __AIENGINE__, __AIEARCH__=10|20|21
  - auto-included intrinsic headers (aiev1/v2/2pintrin.h)
  - ~655 builtins → LLVM intrinsics (~86% direct, ~14% custom lowering)
  - accumulator operator overloads (+/- on v32acc32, v16acc64, v16accfloat)
  │
  v
LLVM IR (with llvm.aie2.*/llvm.aie2p.* intrinsics)
  │
  ├── IR passes with AIE driver defaults:
  │   - vectorizers disabled, mem builtins disabled
  │   - AIE alias analysis, address space flattening
  │   - loop iteration count assumptions enabled
  │
  v
LLVM CodeGen (AIECodeGen)
  ┌─ GlobalISel path (aie2/aie2p) ────────────────────────┐
  │  IRTranslator                                          │
  │    → AIEAddressSpaceFlattening                         │
  │    → AIE2PreLegalizerCombiner (O>0)                    │
  │    → AIEEliminateDuplicatePHI                          │
  │  Legalizer (AIE2LegalizerInfo)                         │
  │    → AIE2PostLegalizerGenericCombiner                  │
  │    → AIEClusterBaseAddress (-aie-address-chaining)     │
  │    → AIEPtrModOptimizer (-aie-global-ptr-mod-opt)      │
  │    → AIE2PostLegalizerCustomCombiner                   │
  │  RegBankSelect                                         │
  │  InstructionSelect (AIE2InstructionSelector)           │
  │    → AIEPostSelectOptimize                             │
  └────────────────────────────────────────────────────────┘
  ┌─ SelectionDAG legacy path (aie1 only) ────────────────┐
  │  AIEISelDAGToDAG                                       │
  └────────────────────────────────────────────────────────┘
  │
  v
Machine IR (target-specific)
  - Frame lowering (upward-growing stack, AIE2FrameLowering)
  - Prologue/epilogue insertion
  - Staged register allocation by family:
      Phase 1: Modifier registers (M)
      Phase 2: 3D dimension registers
      Phase 3: 3D + 2D dimension registers
  - Super-reg rewriter, WAW-reg rewriter, sub-reg constrainer
  - Hardware-loop pseudo expansion (AIEBaseHardwareLoops)
  - Branch/pseudo expansion (AIEPseudoBranchExpansion)
  │
  v
Post-RA Scheduling/Bundling Subsystem (MANDATORY for correctness)
  ┌─ AIEPostRASchedStrategy (extends PostGenericScheduler) ┐
  │  Three-mode dispatch via SchedulingStage:              │
  │    GatheringRegions → Scheduling → SchedulingDone      │
  │                     → Pipelining → PipeliningDone       │
  │                                  → PipeliningFailed     │
  │                                                        │
  │  AIEHazardRecognizer (scoreboard-based):               │
  │    - 9-rule conflict check (FuncUnitWrapper::conflict)  │
  │    - Slot/format + FU + memory bank/object hazards     │
  │    - Alternate descriptor selection under pressure      │
  │                                                        │
  │  Inter-block fixpoint convergence:                     │
  │    - Per-MI latency margins → resource margins         │
  │    - Loop backedge latency/resource verification       │
  │    - Epilogue NOP computation                          │
  │                                                        │
  │  Post-RA software pipeliner (PostPipeliner):           │
  │    - 11 heuristic strategies, 20 runs each            │
  │    - Z3 SAT solver fallback (binary + integer modes)   │
  │    - II retry: ResMII → maxII=40, maxTries=20         │
  └────────────────────────────────────────────────────────┘
  │
  v
Bundle Finalization
  - AIEFinalizeBundle pass
  - applyFormatOrdering() → slot-order normalization
  - Alternate opcode materialization
  - AIEMachineAlignment
  │
  v
MC Layer
  - MCInstLower → MCInst stream
  - Packet encoding (variable-width, nibble-based size dispatch)
  - AIE ELF writer, fixups, relocations
  - Asm printer (semicolon bundle syntax, slot-NOP filling)
  - Asm parser / disassembler
  │
  v
AIE ELF Object + Textual Assembly
```

## 3. Module Dependency Map

### Build-Level Libraries

Topological build order (from `CMakeLists.txt` analysis):

```text
LLVMAIEInfo          (no AIE deps; depends on MC, Support)
LLVMAIEUtils         (no AIE deps; depends on Analysis, CodeGen, Core, Support, TransformUtils)
LLVMAIEAsmPrinter    → LLVMAIEUtils
LLVMAIEDesc          → LLVMAIEInfo + LLVMAIEAsmPrinter (+ MC, Support, TargetParser)
LLVMAIECodeGen       → LLVMAIEInfo + LLVMAIEUtils + LLVMAIEAsmPrinter + LLVMAIEDesc
LLVMAIEAsmParser     → LLVMAIEInfo + LLVMAIEUtils + LLVMAIEDesc (+ MC, MCParser)
LLVMAIEDisassembler  → LLVMAIEInfo + LLVMAIECodeGen (+ MCDisassembler, MC)
```

### Logical Subsystem Dependencies

```text
TableGen ISA/Reg/Sched definitions
  → generated *.inc files (standard + 4 custom AIE generators)
  → consumed by CodeGen + MC components

Clang frontend (AIETargetInfo + builtins + driver)
  → emits LLVM IR with llvm.aie*/llvm.aie2*/llvm.aie2p* intrinsics
  → consumed by AIE CodeGen

AIE CodeGen
  → produces scheduled, bundled MachineInstr stream
  → lowered by MCInstLower → MC emitter

MC layer (Desc/Parser/Printer/Disassembler/ELF)
  → defines binary + textual interface for toolchain and tests

Critical cross-layer dependency:
  Scheduling/bundling depends on MC format metadata (AIEMCFormats)
  for slot sets, conflict sets, and packet-format availability.
```

### Custom TableGen Generators

Beyond upstream LLVM's standard generators:
- `-gen-aie-memory-cycles` — memory access cycle tables
- `-gen-aie-presched-lowering` — pre-scheduling legalization tables
- `-gen-aie-split-instr-tables` — split instruction lookup tables
- `-gen-aie-alternate-itinerary-emitter` — operand-class-keyed itinerary variants

### Boundary APIs

| Boundary | File | Purpose |
|----------|------|---------|
| Pass factory | `AIE.h` | Scheduling, bundling, optimization pass creation |
| AIE2/2P entry | `AIE2.h` | Combiner/selector entry points |
| Frontend target | `clang/.../AIE.h` | Triple, macros, type info |
| Driver/toolchain | `clang/.../ToolChains/AIE.*` | Flags, headers, linker |

## 4. Key Design Decisions

### 1. Slot-First ISA Modeling
Instructions are defined by slot class, then composed into legal packet formats. Slot kinds and widths:

| Slot | AIE2 (bits) | AIE2P (bits) |
|------|------------|-------------|
| lda | 21 | 20 |
| ldb | 16 | 17 |
| alu | 20 | 20 |
| mv | 22 | 22 |
| st | 21 | 20 |
| vec | 26 | 26 |
| lng | 42 | 42 |
| nop | 1 | 1 |

Note: three slot kinds differ between variants (lda: 21→20, ldb: 16→17, st: 21→20). These bit-width differences affect encoding field layouts and format table generation.

Multi-slot pseudos (e.g., `PADD`, `MOV`, `VLD` families) can materialize into any slot that hosts the operation. **Design invariant**: all concrete opcode alternatives for a multi-slot pseudo must have equivalent scheduling itineraries and operand-latency behavior, so slot selection doesn't affect timing correctness.

### 2. Variable-Width Composite Packet Encoding
Packet legality is a slot-set-to-format lookup. AIE2 has 58 concrete composite formats; packets range from 16 to 128 bits. Size dispatch is nibble-based (16 mappings, different per variant — AIE2P remaps odd nibbles to 16-byte packets). `PacketFormats::getFormat(SlotBits)` returns the minimum-size format covering the occupied slots.

### 3. Post-RA Scheduling as Correctness
The hardware does not interlock on most hazards. The compiler constructs a conflict-free schedule. `IssueWidth = 1000` (deliberately high; VLIW format legality is the real constraint). Post-RA scheduling is always enabled and mandatory.

### 4. Signed/Negative Latency Scheduling
Both operands in readiness computation are cast to `int` for signed arithmetic. `BotReadyCycle` can reflect that a predecessor is legally placed earlier than its successor. Lower bound: `-10` (configurable). Liveness is explicitly invalidated after scheduling.

### 5. Nine-Rule Scoreboard Hazard Recognition
`FuncUnitWrapper::conflict()` checks:

| # | Rule | Type |
|---|------|------|
| 1 | Same slot occupied | Bitwise |
| 2 | Same memory bank accessed | Bitwise |
| 3 | Same pointer object accessed | Bitwise |
| 4 | This conflict set ∩ other's slots | Bitwise |
| 5 | Other's conflict set ∩ this slots | Bitwise |
| 6 | Both require same FU | Bitwise |
| 7 | Reserved FU ∩ other's required | Bitwise |
| 8 | Required FU ∩ other's reserved | Bitwise |
| 9 | Combined slots have no valid format | Format lookup |

Rules 1–8 are O(1) bitwise. Rule 9 is the definitive format legality check.

### 6. Alternate Descriptor Selection During Scheduling
During `getHazardType()`, instructions with alternate opcodes are tested in sequence. The first opcode producing `NoHazard` is recorded via `setAlternateDescriptor()`. This allows adapting instruction encoding to slot/resource pressure without changing semantics. The choice is preserved through emission.

### 7. Two-Level Software Pipelining

| Level | Framework | When Used |
|-------|-----------|-----------|
| Pre-RA | Upstream `MachinePipeliner` | Simple loops, ≤3 stages, adequate register pressure |
| Post-RA | Custom `PostPipeliner` | Complex loops deferred by pre-RA, low-II high-stage loops |

Post-RA details:
- 11 heuristic strategies with 8 discriminators (`NodeNum`, `Latest`, `Earliest`, `Critical`, `Sibling`, `LCDLatest`, `DepLength`, `Liveness`)
- 20 convergence runs per heuristic
- Z3 SAT solver fallback: binary formulation (`NInstr × NStages × II` variables) and integer formulation (`2N` variables)
- Solver timeout: 2000ms, deterministic pre-estimate: `0.02 × Rows × Columns`
- II retry: starts at `ResMII`, increments up to `maxII=40` or `maxTries=20`

Pre-RA defers to post-RA when: stage count > 3 and II < 11, or any loop-carried latency ≥ II.

### 8. GlobalISel-First Architecture for Modern Targets
`aie2`/`aie2p` use staged GlobalISel with custom combiners instead of monolithic DAG lowering. This is a **hard architectural boundary**: AIE2P has no SelectionDAG path at all (no dag-isel or pseudo-lowering TableGen generators). AIE1 uses SelectionDAG exclusively with no GlobalISel support.

The GlobalISel pipeline has 6 stages (3 custom combiners + legalizer + analysis + post-select):
1. `AIE2PreLegalizerCombiner` — pattern-matching combiner (disabled at O0)
2. Legalizer (`AIE2LegalizerInfo`) — type/operation legalization
3. `AIE2PostLegalizerGenericCombiner` — generic post-legal combines
4. `AIEClusterBaseAddress` — address chaining analysis pass
5. `AIE2PostLegalizerCustomCombiner` — AIE-specific post-legal combines
6. `AIEPostSelectOptimize` — post-instruction-selection cleanup

### 9. Operand-Class-Sensitive Scheduling
The same operation gets different resources and latencies depending on operand register class. This is modeled via `ItineraryRegPairs` in the schedule model. AIE2P's 4,010 itinerary classes (vs AIE2's 278) reflect this fine-grained sensitivity. Bypass annotations (`MOV_Bypass`, `VEC_Bypass`, `MV_Bypass`) modify effective timing.

### 10. Builtin-Heavy Frontend Contract
~655 Clang builtins map to backend intrinsics (AIE1: ~41, AIE2: ~395, AIE2P: ~219 — each variant's builtins are **separate namespaces**, not cumulative). ~86% are direct 1:1 mappings via `getAIE{1,2,2P}IntrinsicFunction()`. ~14% (~90 builtins) require custom lowering across 6 patterns:

| Pattern | Builtins | Example |
|---------|---------|---------|
| Address manipulation (i32↔i20 trunc/ext) | `add_2d`, `add_3d` | Truncate input increments i32→i20, zext output counts i20→i32 |
| Multi-result tuple unpacking | ~40 | `vabs_gtz8` → vector result + condition mask via output pointer |
| Sparse operations (triple output) | ~36 | `sparse_pop_4` → vector data + mask + pointer |
| BFP16 mantissa/exponent splitting | 3 | `v64accfloat_to_v64bfp16ebs8` → mantissa + exponent |
| FIFO operations (state tracking) | ~26 | `fifo_ld_pop` → data + pointer + state + position |
| Cascade stream expand | 2 | `scd_expand_ACC1024_incr` → accumulator + updated position |

Custom accumulator types (`__acc32`, `__acc64`, `__accfloat`) support operator overloads for natural C++ syntax. BFloat16 is supported on AIE2/AIE2P only.

## 5. Delta from Upstream LLVM

### Reused Infrastructure (with AIE specialization)
- Standard target layering: `TargetMachine`, `Subtarget`, `InstrInfo`, `FrameLowering`, MC components
- GlobalISel framework stages (IRTranslator, Legalizer, RegBankSelect, InstructionSelect)
- `ScheduleDAGMI` → extended with 3-mode dispatch (`GatheringRegions`, `Pipelining`, `Scheduling`)
- `PostGenericScheduler` → extended with 6 additional heuristics
- `MachinePipeliner` → defers complex loops to post-RA pipeliner
- `ScheduleHazardRecognizer` → extended with scoreboard, memory bank/object, and format checks
- SelectionDAG path retained for legacy `aie1`

### Custom-Built (Not in Upstream)

**Scheduling/bundling** (4 major subsystems):
- `AIEHazardRecognizer` with `ResourceScoreboard<FuncUnitWrapper>` and 9-rule conflict model
- `AIEInterBlockScheduling` with fixpoint convergence and per-MI margin tracking
- `AIEPostPipeliner` with 11 heuristic strategies and Z3 solver integration
- `AIE::Bundle<I>` with slot-set-to-format legality model

**CodeGen passes** (11 AIE-specific):
- 3 pre-legalizer passes (address space flattening, combiner, duplicate PHI elimination)
- 3 post-legalizer passes (generic combiner, cluster base address, pointer-mod optimizer)
- 1 post-select pass
- 4 late-pipeline passes (finalize bundle, machine alignment, spill slot optimization, split instruction rewriter)

**RA helper passes** (7):
- Super-reg rewriter, WAW-reg rewriter, sub-reg constrainer, reg-class constrainer
- Tied-reg operand handling, unallocated super-reg rewriter, live-reg tracking

**Custom TableGen generators** (4):
- `aie-memory-cycles`, `aie-presched-lowering`, `aie-split-instr-tables`, `aie-alternate-itinerary-emitter`

**MC layer**:
- Composite packet encoding with nibble-based size dispatch
- Custom fixup-field-to-relocation mapping per slot
- Per-variant AsmBackend, MCCodeEmitter, MCFormats

**Frontend**:
- ~655 builtins across 3 variants with custom emission handlers
- Custom accumulator types and operator overloads
- AIE-specific ABI classification (sparse type alignment, accumulator InReg)

### Modified vs. Replaced
- Mostly **extension/override** of upstream frameworks
- Effectively **behavioral replacement** for: late scheduling/bundling legality, inter-block fixpoint convergence, post-RA software pipelining, and frontend builtin lowering paths

## 6. Build Sequence (Dependency-Ordered)

### Phase 1: Target Identity and MC Base
1. `LLVMAIEInfo` — triple/target registration (`aie`, `aie2`, `aie2p`)
2. `LLVMAIEUtils` — shared analysis helpers (loop utils, MBB utils, IR utils)
3. `LLVMAIEAsmPrinter` — print support used by MC descriptors
4. `LLVMAIEDesc` — MC formats, fixups, ELF descriptors, MCTargetDesc

### Phase 2: ISA/TableGen Foundation
1. Define base `.td` files: `AIEBaseInstrInfo.td`, `AIEBaseRegisterInfo.td`, `AIEBaseInstrFormats.td`, `AIEBaseRegisterBanks.td`
2. Add AIE2 overlays: `AIE2.td`, `AIE2RegisterInfo.td`, `AIE2InstrInfo.td`, `AIE2Schedule.td`, `AIE2Slots.td`, `AIE2CallingConv.td`
3. Add AIE2P overlays: parallel structure under `aie2p/`
4. Enable standard TableGen generators + 4 custom AIE generators
5. Integrate generated `*.inc` headers

**TableGen generator counts per variant:**
- AIE1: ~12 standard generators
- AIE2: ~12 standard + 4 custom + 3 GISel combiner generators
- AIE2P: ~10 standard + 4 custom + 3 GISel combiner generators (no dag-isel, no pseudo-lowering)

### Phase 3: Core CodeGen Bring-Up
1. `AIEBaseTargetMachine` / `AIE2TargetMachine` — pass pipeline registration
2. `AIEBaseSubtarget` / `AIE2Subtarget` — feature/scheduling model binding
3. `AIEBaseFrameLowering` / `AIE2FrameLowering` — upward-growing stack model
4. `AIECallLowering` — calling convention (pointer, scalar, vector, accumulator classes)
5. GlobalISel pipeline: `AIE2LegalizerInfo`, `AIE2RegisterBankInfo`, `AIE2InstructionSelector`
6. AIE combiner stack: 3 pre/post-legalizer combiners + post-select optimizer
7. `AIEBaseHardwareLoops` — zero-overhead loop conversion
8. Legacy `aie1` SelectionDAG path (`AIEISelDAGToDAG`) if required

### Phase 4: Register Allocation and Machine Legalization
1. Staged RA by register family: modifier → 3D dim → 3D+2D dim
2. Super-reg rewriter, WAW-reg rewriter, sub-reg constrainer
3. Tied-operand constraint handling and verifier invariants
4. Hardware-loop and branch/pseudo expansion
5. Spill slot optimization (`AIESpillSlotOptimization`)

### Phase 5: Scheduling and Bundling (Critical Path)
1. `AIEHazardRecognizer` — scoreboard + 9-rule `FuncUnitWrapper::conflict()`
2. `AIE::Bundle<I>` — slot-set-to-format legality model with 7-step `canAdd()`
3. `AIEAlternateDescriptors` — schedule-time opcode adaptation
4. `AIEPostRASchedStrategy` — signed-latency scheduling, bottom-up DeltaCycles
5. `AIEInterBlockScheduling` — fixpoint convergence with per-MI margins
6. `AIEPostPipeliner` — modulo scheduling with 11 heuristics + Z3 solver
7. `AIEMultiSlotInstrMaterializer` — static MSP assignment for loops
8. `AIEFinalizeBundle` + `applyFormatOrdering()` — final bundle normalization
9. `AIEMachineAlignment` — alignment constraints

### Phase 6: MC Tool Surfaces and Frontend
1. Per-variant MCCodeEmitter (`AIE2MCCodeEmitter`, `AIE2PMCCodeEmitter`)
2. Per-variant AsmBackend (`AIE2AsmBackend`, `AIE2PAsmBackend`)
3. Asm parser / disassembler with bundle syntax support
4. Relocation/fixup correctness and round-trip validation
5. Clang `AIETargetInfo` + `AIEToolChain` + builtin emission
6. Stabilize against CodeGen/MC/Driver test suites

**Strict library topological order:**
1. `LLVMAIEInfo`
2. `LLVMAIEUtils`
3. `LLVMAIEAsmPrinter`
4. `LLVMAIEDesc`
5. `LLVMAIECodeGen`
6. `LLVMAIEAsmParser`
7. `LLVMAIEDisassembler`

## 7. Cross-Cutting Concerns

### 1. VLIW Affects ISA, Scheduler, and MC Equally
Slot definitions, bundle legality (`canAdd()` 7-step check), textual `;` packet syntax, and binary packet encoding must agree. The `AIEMCFormats` interface is consumed by both CodeGen (hazard recognizer) and MC (emission). Changes to slot definitions or format tables propagate to scheduling, assembly output, and binary encoding simultaneously.

### 2. Scheduling Decisions Affect Emitted Opcodes
Alternate descriptor choices made during `getHazardType()` are recorded in `AIEAlternateDescriptors` and must be preserved through `applyFormatOrdering()` and MCInst lowering. If the alternate selection is lost, the emitted instruction may occupy the wrong slot or violate format constraints.

### 3. Register Model Affects Nearly Every Stage
Calling convention, regbank selection, RA rewrite passes, verifier checks, and scheduling pressure all depend on register class topology.

**Callee-saved register split** (intentional ABI difference):
- AIE2: `lr, r16-r23, p6, p7` (11 registers)
- AIE2P: `lr, r8-r15, p6, p7` (10 registers, **different GPR range**)
This difference affects prologue/epilogue generation, CSR save/restore ordering, and RA pressure.

**Register bank differences**:
- AIE2: `GPRRegBank`, `PtrRegBank`, `VecRegBank`, `AccRegBank [ACC256, ACC512, ACC1024]`
- AIE2P: `GPRRegBank` (includes EXPVEC64), `PtrRegBank`, `VecRegBank`, `AccRegBank [ACC512, ACC1024, ACC2048]`, `FifoRegBank` (new)
- Register class drives itinerary selection (4,010 variants on AIE2P)
- AIE2P adds `AIEUnallocatedSuperRegRewriter` (fine-grained mode) to the RA pipeline

**Frame lowering variant differences**:
- AIE2: SP adjustment granularity = 32 bytes, 17-bit immediate range → ±4MB effective stack
- AIE2P: SP adjustment granularity = 64 bytes, 18-bit immediate range → ±16MB effective stack; MaxVectorAlign = 512 bits (vs AIE2's 256 bits)
- AIE2P adds conservative emergency scavenging slots when estimated stack exceeds ~4KB (signed 12-bit immediate range) for `eP` and `eDJ` register classes

### 4. Address-Space/Bank Semantics Cross Frontend and Backend
Clang-side address-space annotations → backend memory-bank bits → hazard recognizer memory conflict checks. Bank conflicts are checked at memory access cycles (not issue cycles). The `MemoryObjectEnumerator` tracks up to 64 pointer base objects; disabled in pre-RA to avoid over-constraining.

### 5. Loop Transformations Span IR→MIR→Post-RA
- IR: loop pragmas (`aie.loop.tripcount.min`), hardware-loop detection
- MIR: `AIEBaseHardwareLoops` pseudo expansion, multi-slot pseudo materialization
- Post-RA: inter-block fixpoint scheduling, post-RA software pipelining, trip count adjustment (`-(NStages-1)`)
- Epilogue analysis: NOP insertion based on max(latency hazards, resource conflicts)

### 6. MC Binary Interface Constrains Frontend and Backend
Builtin lowering and instruction selection must produce forms encodable into AIE packet formats. Fixup fields are slot-specific; relocations map to fixup kinds per slot position. The MC layer defines the contract that both CodeGen emission and external tools (linker, debugger) must satisfy.

### 7. Tests Encode Behavioral Specification
1,855 test files (1,391 MIR, 432 LLVM IR) assert exact bundle order, instruction shape, relocation kinds, and diagnostic text. These are de facto architecture contracts. By variant: aie2 (821), aie2p (541), GlobalISel (271), aie1 (76), shared (119). Test categories:
- MIR hazard/scheduling tests (cycle-accurate assertions)
- Binary encoding round-trip tests
- ABI/calling-convention tests
- Loop pipelining result tests
- Driver flag tests
Known limitations: 4 explicit XFAIL files (AIE1 only), 10 xfail-intent files (GlobalISel unsupported operations).

### 8. Scheduling Fixpoint Is a State Machine
The inter-block scheduling algorithm is not a single pass but an iterative state machine:
```
GatheringRegions → Scheduling ⟲ (re-schedule with adjusted margins)
                 → SchedulingDone → Pipelining ⟲ (increment II, retry)
                                   → PipeliningDone
                                   → PipeliningFailed
```
State tracked in `FixedpointState`: `LatencyMargin`, `PerMILatencyMargin`, `PerMIExtraDepth`, `ResourceMargin`, `II`, `IITries`, `MaxLatencyExtent`, `MaxResourceExtent`, `NumIters`.

## 8. Gaps and Open Questions

### Known Documentation Gaps

1. **Register-layer synthesis missing** — `docs/aie-spec/04-register/` is empty. Register subsystem has partial coverage via ISA/codegen/scheduling docs but no dedicated synthesis covering:
   - Register bank info implementation
   - Staged RA pass ordering and family decomposition
   - Super-reg/WAW rewriter algorithms
   - Tied-operand constraint verification

2. **Pass-option matrix** — 50+ AIE-specific command-line options exist across driver and backend. No single reference matrix by optimization level and target variant.

3. **MC fixup-to-relocation reference** — overview exists but a complete opcode→fixup→relocation mapping table per slot is not synthesized.

4. **AIE1 vs AIE2/AIE2P maintenance boundary** — both paths exist (SelectionDAG vs GlobalISel-first). Parity strategy and long-term direction not documented.

### Resolved Questions

These were initially gaps but are now documented in the spec:

- **Slot assignment for multi-slot pseudos**: `SlotMapping` heuristic with bank→slot reuse, unused-slot fallback, LRU rotation. Documented in `vliw-bundling.md`.

- **Format legality algorithm**: `PacketFormats::getFormat(SlotBits)` returns minimum-size format covering slots. Two-stage check: fast conflict-set rejection (O(1) bitwise AND), then format-table lookup. Documented in `vliw-bundling.md`.

- **Memory bank conflict modeling**: conservative for performance (hardware can stall), not for correctness. Pointer object tracking via `MemoryObjectEnumerator` with 64-object limit, disabled in pre-RA. Documented in `hazard-recognizer.md`.

- **Scoreboard depth**: `max(2 × PipelineDepth, UserScoreboardDepth)` rounded to power of 2. Pre-RA default: 128 cycles. Documented in `hazard-recognizer.md`.

- **Builtin custom lowering patterns**: 6 patterns covering ~90 builtins (address truncation, tuple unpacking, sparse triple outputs, BFP16 splitting, FIFO state tracking, cascade expand). Documented in `builtins.md`.

- **Pipeliner deferral criteria**: Pre-RA defers to post-RA when stage count > 3 and II < 11, or any loop-carried latency ≥ II. Does not defer if II ≥ 4. Documented in `software-pipelining.md`.

### Open Technical Questions

5. **Quantitative scheduler acceptance criteria** — mechanisms are documented, but explicit performance thresholds for "good enough" schedules are not consolidated.

6. **AIE2P immediate print/parse asymmetry** — potential immediate-prefix mismatch risk between asm printer and parser. Needs focused round-trip testing.

7. **MC parser directive handling** — `AIEBaseAsmParser::ParseDirective` is a stub; directive behavior depends on generic handling/streamer side. Unclear if this is intentional or incomplete.

## Appendix: Calling Convention Summary

### Argument-Passing Registers

| Class | AIE2 | AIE2P |
|-------|------|-------|
| Pointer (20-bit) | p0–p5 | p0–p5 |
| Scalar (32-bit) | r0–r7 | r0–r7 |
| Long (64-bit) | l0–l7 | l0–l15 |
| Vector (256-bit) | wl, wh | wl, wh |
| Vector (512-bit) | x0–x11 | x0–x11 |
| Vector (1024-bit) | y2–y5 | y2–y5 |
| Accumulator (2048-bit) | — | dm0–dm4 |

### Callee-Saved Registers

| Variant | Registers |
|---------|-----------|
| AIE2 | `lr, r16-r23, p6, p7` |
| AIE2P | `lr, r8-r15, p6, p7` |

Note: AIE2 and AIE2P use **different GPR ranges** for callee-saved registers.

## Appendix: Key Command-Line Options

### Scheduling

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-negative-latencies` | `true` | Enable signed-latency scheduling |
| `-aie-neglatency-lowerbound` | `-10` | Lower bound for negative latency deps |
| `-aie-scoreboard-depth` | `128` | Pre-RA scoreboard depth |
| `-aie-loop-aware` | `true` | Iterative single-block loop scheduling |
| `-aie-interblock-scoreboard` | `true` | Cross-block scoreboard initialization |
| `issue-limit` | `6` | Max instructions per cycle |

### Software Pipelining

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-postpipeliner-maxii` | `40` | Maximum II to attempt |
| `-aie-postpipeliner-maxtry-ii` | `20` | Maximum II steps |
| `-aie-postpipeliner-heuristic-runs` | `20` | Convergence runs per heuristic |
| `-aie-postpipeliner-solver-timeout` | `2000` | Z3 timeout (ms) |
| `-aie-pipeliner-max-stagecount` | `3` | Max stages before pre-RA rejection |
| `-aie-postpipeliner-limit` | `4` | II threshold for post-RA preference |

### Memory Hazards

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-addrspace-none-is-safe` | `true` | Address space 0 is conflict-free |
| `-aie-recognize-pointer-hazards` | `true` | Pointer-object hazard tracking |
