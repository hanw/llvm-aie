# Instruction Catalog (Structural)

Sources:
- `llvm/lib/Target/AIE/AIE2InstrInfo.td`
- `llvm/lib/Target/AIE/AIE2GenFixupInstrInfo.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PInstrInfo.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PGenInstrInfo.td`
- `llvm/lib/Target/AIE/AIE2InstrPatterns.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PInstrPatterns.td`
- `llvm/lib/Target/AIE/AIEBaseInstrInfo.td`
- `llvm/lib/Target/AIE/AIEBaseInstrPatterns.td`
- `llvm/lib/Target/AIE/AIEInstrGISel.td`
- `llvm/lib/Target/AIE/AIECombine.td`
- `llvm/lib/Target/AIE/aie1/AIE1InstrInfo.td`

Goal: summarize ISA families and backend design intent (not exhaustively list all mnemonics).

## 1. Scalar ALU and Integer Core Ops

### Representative Operations

| Category | Instructions |
|----------|-------------|
| Arithmetic | `add`, `add.nc`, `sub`, `adc`, `sbc`, `mul` |
| Logic/bit | `and`, `or`, `xor`, `abs`, `clb`, `clz` |
| Compare/select | `eq`, `ne`, `lt`, `ltu`, `ge`, `geu`, `eqz`, `nez`, `sel.eqz`, `sel.nez` |
| Shifts | `ashl`, `lshl` |
| Extensions | `ext_8` (zero/sign extend 8-bit), `ext_16` (zero/sign extend 16-bit) |

### Typical Operands

- Scalar registers: `eR`, constrained subsets (`eR26`, `eR27`, etc.)
- Immediates: signed/unsigned/scaled classes (`simm7`, `c7s`, `c10s_step64`)
- Status register side effects: `srCarry` for add/sub with carry

### Design Notes

- Many scalar ops have variant-specific operand-class constraints, not just plain GPR operands.
- Some mnemonics exist in multiple slot forms (ALU vs MV forms), exposed via generated defs and multi-slot pseudo strategy.
- Compare patterns use multiclass expansion (`CmpPat`, `CmpZeroPat`, `CmpSwapPat`) to cover all condition code variants systematically.

## 2. Vector and Accumulator Operations

### Representative Families

| Category | Instructions |
|----------|-------------|
| Vector arithmetic | `vadd`, `vsub`, `vneg`, `vmax*`, `vmin*`, `vabs*` |
| Multiply/accumulate | `vmul`, `vmac`, `vmsc`, `vaddmac`, `vnegmul`, `vnegmac` |
| Lane/data motion | `vextract`, `vinsert`, `vshuffle`, `vshift`, `vbcst`, `vextbcst`, `vmov` |
| Format conversion/pack | `vconv.*`, `vpack.*`, `vunpack.*`, `vsrs.*`, `vups.*` |

### Typical Operands

- Vectors: `VEC128/256/512/1024`, sparse/BFP/FIFO-related vector classes (AIE2P)
- Accumulators: `ACC256/512/1024` (AIE2), `ACC512/1024/2048` (AIE2P)
- Mode/config operands encoded through dedicated fields and/or pseudo configuration nodes

### Vector Configuration System

Vector operations use a rich configuration encoding:
- **AMODE** (accumulator width): I32 (0), I64 (1), FP32 (2)
- **BMODE** (multiplication precision): 8x4, 8x8, 16x8, 16x16, 32x16
- **Dynamic control bits**: `dynZeroAccum`, `accShift`, `dynMulNeg`, `dynAcc0Neg`, `dynAcc1Neg`, `dynTermNeg`

These configuration fields are encoded as immediate operands within the VEC slot payload.

### Design Notes

- Vector math is tightly coupled to register substructure and mode fields (`amode/bmode/cmode`, sign/rounding controls).
- AIE2P extends to richer mixed-width and BFP-related vector/accumulator forms, causing a large generated opcode space.
- `UPPER_SRS` and `UPS_UNIT` resources are dedicated FuncUnits in the schedule model, reflecting that SRS (shift-round-saturate) and UPS (upshift) are shared hardware resources.

## 3. Memory Operations

### Scalar Load/Store

| Category | Instructions |
|----------|-------------|
| Scalar loads | `lda*` variants (`LDA_*`) |
| Scalar stores | `st*` variants (`ST_*`) |

### Vector Load/Store

| Category | Instructions |
|----------|-------------|
| Vector loads | `vlda*`, `vldb*`, including `2d`, `3d`, post-increment, sparse/compressed, `4x*` |
| Vector stores | `vst*`, including `srs`, `pack`, `conv`, flush/push/pop (AIE2P) |

### Addressing Mode Architecture

This is a core architectural differentiator. Addressing modes are defined as explicit classes with typed sub-operands:

| Mode | Syntax Pattern | Description |
|------|---------------|-------------|
| `ag_idx` | `*(ptr + dj)` | Indexed addressing via jump register |
| `ag_idx_imm` | `*(ptr + imm)` | Indexed with immediate offset |
| `ag_pstm_nrm` | `t = *ptr; ptr += m; return t` | Post-increment with modifier register |
| `ag_pstm_nrm_imm` | `t = *ptr; ptr += imm; return t` | Post-increment with immediate |
| `ag_pstm_2d` | `ptr += add2d(d)` | 2D addressing with dimension register |
| `ag_pstm_3d` | `ptr += d` (3D) | 3D addressing with dimension register |
| `ag_spill` | `*(sp + imm)` | Stack pointer + immediate (for spills) |

Address mode classes:
- `AIE2_ag_ptr_imm3/4/6/7/10`: pointer + immediate with 3–10 bit widths
- `AIE2_agb_ptr_imm9`: special "B" variant with 9-bit immediate
- `AIE2_ag_ptr_mod`: pointer + modification register
- `AIE2_ag_ptr_dj`: pointer + delay jump register

Sub-mode distinction:
- `ag_all`: full addressing mode set (indexed, post-increment, 2D/3D, spill)
- `ag_nospill`: for sub-word loads/stores that cannot use spill mode

### Spill-Specialized Pseudos

Dedicated `*_spill` pseudo instructions exist for scalar, vector, accumulator, and FIFO register classes. These encode stack-relative access patterns optimized for register spilling.

### Typical Operands

- Pointers/modifiers: `eP`, `eM`, `eD`, `eDS`, `eDJ`, `eDN`, `eDC`
- Scalar/vector data classes selected by instruction form
- Step-scaled immediates for aligned memory access: `immx4` (4-byte), `immx32` (32-byte), `immx128` (128-byte)

### Design Notes

- Memory ops are not generic base+offset only; multidimensional and data-transform memory pipelines are explicit ISA features.
- The dual load paths (LDA/LDB) enable two simultaneous memory reads per cycle in a VLIW bundle.
- Vector stores can incorporate data transformation (SRS, pack, convert) as part of the store operation itself.

## 4. Control Flow

### Representative Operations

| Category | Instructions |
|----------|-------------|
| Branch/jump | `j`, `jz`, `jnz`, `jnzd`, `jl` (jump-and-link), indirect forms |
| Return/call | `PseudoRET`, `PseudoJL`, stack-adjust pseudos |
| Loop control | `loop_start`, `LoopJNZ` (mapped to hardware loop sequences) |
| Slot-specific nops | `nop`, `nopa`, `nopb`, `nopx`, `nopm`, `nops`, `nopv` |

### Design Notes

- Control-flow instructions are closely integrated with scheduling barriers and delay-slot expansion pseudos.
- Hardware loop support (`loop_start`/`LoopJNZ`) avoids branch overhead for counted loops, a common pattern in DSP and ML workloads.
- Per-slot nops (`nopa` for LDA, `nopb` for LDB, etc.) are needed to fill unused slots in VLIW bundles.

## 5. Special / System / Synchronization

### Representative Operations

| Category | Instructions |
|----------|-------------|
| Synchronization | `acq`, `acq.cond`, `rel`, `rel.cond` |
| Lifecycle/events | `done`, `event`, `sched_barrier` |
| Architecture-specific | Status/control register interactions via constrained register classes |

### Design Notes

- Synchronization and system ops are explicitly modeled as side-effecting and receive dedicated scheduling treatment.
- `acq`/`rel` (acquire/release) operations interact with the `SEMAPHORE` functional unit in the schedule model, with 4-cycle latency.
- `sched_barrier` is a compiler directive that prevents instruction reordering across the barrier.

## 6. Operand-Type Patterns (Representative)

Common operand domains seen in instruction defs/patterns:

| Domain | Classes |
|--------|---------|
| Scalar | `eR`, `eL` |
| Pointer/modifier | `eP`, `eM`, `eD`, `eDS`, `eDJ`/`eDN`/`eDC` |
| Vector | `VEC128`, `VEC256`, `VEC512`, `VEC1024` |
| Accumulator | `ACC256`, `ACC512`, `ACC1024`, `ACC2048` (AIE2P) |
| Masks/sparse/fifo/exp | AIE2P-specific classes |
| Unsigned immediates | `immx2`..`immx20` (by bit width) |
| Signed immediates | `simm3`..`simm32` |
| Scaled immediates | `immx4`, `immx8`, `immx16`, `immx32`, `immx64`, `immx128` |

## 7. Structural Takeaway

The instruction set is best understood as an interaction of:
- **Slot-typed encoding families** — each instruction belongs to exactly one functional slot
- **Strongly constrained operand register classes** — operand position determines legal register subset
- **Schedule/resource-aware variants** — itinerary selection depends on concrete operand types

That combination is the core design difference from a conventional scalar RISC ISA catalog.

## 8. Generic-MIR Instruction Layer (Selection-Side ISA View)

`AIEInstrGISel.td` adds AIE-specific generic machine ops that act as an intermediate ISA vocabulary before final opcode selection:

| Category | Instructions |
|----------|-------------|
| Post-increment loads | `G_AIE_POSTINC_LOAD`, `G_AIE_POSTINC_ZEXTLOAD`, `G_AIE_POSTINC_SEXTLOAD` |
| Offset loads | `G_AIE_OFFSET_LOAD`, `G_AIE_OFFSET_ZEXTLOAD`, `G_AIE_OFFSET_SEXTLOAD` |
| 2D/3D loads | `G_AIE_POSTINC_2D_LOAD`, `G_AIE_POSTINC_3D_LOAD` |
| Post-increment stores | `G_AIE_POSTINC_STORE`, `G_AIE_POSTINC_TRUNCSTORE` |
| Offset stores | `G_AIE_OFFSET_STORE`, `G_AIE_OFFSET_TRUNCSTORE` |
| 2D/3D stores | `G_AIE_POSTINC_2D_STORE`, `G_AIE_POSTINC_3D_STORE` |
| Vector element ops | `G_AIE_BROADCAST`, `G_AIE_PAD_VECTOR`, `G_AIE_UNPAD_VECTOR` |
| Vector shuffles | `G_AIE_VSHUFFLE`, `G_AIE_VSELECT` |

`AIECombine.td` defines target combine rules that canonicalize these generic ops into backend-preferred shapes.

### Design Implication

The effective ISA contract in this backend has two layers:
1. **Generic AIE gMIR ops** for legalization/combining — these capture addressing patterns and data movement abstractly
2. **Concrete slot-bound VLIW opcodes** for final encoding — these are the per-slot instruction definitions

The ISel patterns in `AIE2InstrPatterns.td` and `AIE2PInstrPatterns.td` bridge these layers, with multiclass expansion for systematic coverage of ALU ops (`PatGpr`, `PatGprGpr`), compare/select variants, and vector configuration.
