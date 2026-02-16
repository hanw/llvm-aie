# AIE Instruction Selection Patterns

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEISelDAGToDAG.h`
- `llvm/lib/Target/AIE/AIEISelDAGToDAG.cpp`
- Generated matcher: `AIEGenDAGISel.inc`
- Related pass factory declaration: `llvm/lib/Target/AIE/AIE.h` (`createAIEISelDag`)
- GlobalISel instruction selector: `llvm/lib/Target/AIE/AIE2InstructionSelector.cpp`
- GlobalISel instruction selector: `llvm/lib/Target/AIE/aie2p/AIE2PInstructionSelector.cpp`
- TableGen patterns: `llvm/lib/Target/AIE/AIEBaseInstrPatterns.td`, `llvm/lib/Target/AIE/AIE2InstrPatterns.td`, `llvm/lib/Target/AIE/aie2p/AIE2PInstrPatterns.td`
- Legalizer info: `llvm/lib/Target/AIE/AIE2LegalizerInfo.*`, `llvm/lib/Target/AIE/aie2p/AIE2PLegalizerInfo.*`

## Two ISel Paths

| Path | Targets | Framework | Key File |
|------|---------|-----------|----------|
| SelectionDAG | AIE1 (legacy) | `SelectionDAGISel` | `AIEISelDAGToDAG.cpp` |
| GlobalISel | AIE2, AIE2P | `InstructionSelector` | `AIE2InstructionSelector.cpp`, `AIE2PInstructionSelector.cpp` |

The production path for AIE2/AIE2P is GlobalISel. There is no SelectionDAG fallback for these targets.

## SelectionDAG Path (AIE1 Legacy)

### Public Interface
`AIEDAGToDAGISel`:
- `Select(SDNode *Node)`
- `SelectFrameIndex(SDValue &N, SDValue &R)`

Legacy pass wrapper:
- `createAIEISelDag(AIETargetMachine &TM)`

### Selection Strategy
- Main matching is TableGen-driven (`SelectCode(Node)` via `AIEGenDAGISel.inc`).
- Hand-written selector logic is intentionally small (~88 lines) and targeted.

### Custom Select Methods
`Select(SDNode *Node)` custom logic:
- Fast path for already-selected machine opcodes.
- Special-case constant selection: `i32` zero constants are forced to explicit `MOV_U20 0` form, canonicalizing null/zero materialization to a predictable target opcode.

`SelectFrameIndex(SDValue &N, SDValue &R)`:
- Accepts only `ISD::FrameIndex` nodes.
- Converts to target frame index operands (`TargetFrameIndex`) for addressing patterns.

### Complex Pattern Matching
- Complex patterns are declared in TableGen (`ComplexPattern<>` references in instruction definitions) and routed to helpers such as `SelectFrameIndex`.
- The hand-written selector is not a large custom matcher; it is a bridge for a few target-specific operand forms.

## GlobalISel Path (AIE2/AIE2P Production)

### Architecture

The GlobalISel pipeline for AIE2/AIE2P consists of four stages:

```
IR → IRTranslator → gMIR
  → Legalizer (AIE2/AIE2PLegalizerInfo)
  → RegBankSelect (AIE2/AIE2PRegisterBankInfo)
  → InstructionSelect (AIE2/AIE2PInstructionSelector)
```

### Legalizer

The legalizer defines which operations are legal, need widening, need lowering, or need custom handling. Key policies:

**Scalar operations:**
- `i32` is the native scalar width; narrower types are widened.
- `i64` operations are lowered (split to 32-bit pairs where possible).
- No hardware division/remainder — these are libcalled.

**Vector operations:**
- Legal vector types correspond to register classes: 256-bit, 512-bit, 1024-bit.
- 128-bit vectors are widened to 256-bit (matching `getPreferredVectorAction() = TypeWidenVector`).
- AIE2P additionally legalizes 2048-bit accumulator operations.

**Memory operations:**
- Load/store legality follows register class widths.
- Post-increment and 2D/3D addressing patterns are expressed as target-specific generic opcodes (`G_AIE_POSTINC_LOAD`, etc.) during combining, then selected to concrete instructions.

### Combiner Pipeline (Pre-Selection)

Three combiner passes run between legalization and selection:

1. **Pre-legalizer combiner** (`AIE2PreLegalizerCombiner`):
   - Canonicalizes generic MIR before legalization.
   - Includes custom intrinsic and shuffle/vector sequence combining.

2. **Post-legalizer generic combiner** (`AIE2PostLegalizerGenericCombiner`):
   - TableGen-driven and helper-driven generic-MI combines after legalization.

3. **Post-legalizer custom combiner** (`AIE2PostLegalizerCustomCombiner`):
   - Applies target-specific combines, including pointer-modifier optimizations.
   - Consumes `AIEPtrModOptimizer` analysis results when enabled.
   - Forms post-increment load/store patterns, 2D/3D addressing patterns.

### Instruction Selector

The instruction selector (`AIE2InstructionSelector` / `AIE2PInstructionSelector`):
- Uses TableGen-generated match tables as the primary selection mechanism.
- Custom selection for operations not expressible in TableGen patterns.
- Created in subtarget constructor via `createAIE2InstructionSelector()` / `createAIE2PInstructionSelector()`.

### Post-Select Optimization

`AIEPostSelectOptimize` runs immediately after instruction selection to:
- Fold subreg-copy chains into super-reg copies.
- Rewrite `INSERT_SUBREG` chains into `REG_SEQUENCE`.
- Remove redundant reserved-register copies/immediates.
- Duplicate addressing registers to reduce downstream conflicts.
- Fix memory operand info for loads using non-pointer operands.
- AIE2P-specific: store-flush conversion pattern (`VST.FLUSH` + conversion chain adjustment).

## TableGen Pattern Structure

### Scalar ALU Patterns
Multiclass expansions provide systematic coverage:
- `PatGpr` — single GPR operand patterns
- `PatGprGpr` — two-GPR operand patterns
- Compare/select variants via `CmpPat`, `CmpZeroPat`, `CmpSwapPat`

### Address Mode Patterns
Address mode selection is encoded in pattern classes:
- `ag_idx` patterns — indexed addressing `*(ptr + dj)`
- `ag_pstm_nrm` patterns — post-increment `t = *ptr; ptr += m`
- `ag_pstm_2d`/`ag_pstm_3d` patterns — multidimensional addressing
- `ag_spill` patterns — stack-relative `*(sp + imm)`

### Vector Configuration Patterns
Vector operations encode mode bits in patterns:
- `AMODE` (accumulator width): I32, I64, FP32
- `BMODE` (multiplication precision): 8x4, 8x8, 16x8, 16x16, 32x16
- Dynamic control bits encoded as immediate operands

## Notable Pattern-Matching Decisions
- Keep manual matching minimal; rely on declarative TableGen for most patterns.
- Canonicalize zero constants early to reduce variation in emitted machine IR.
- Treat frame-index selection as a complex operand matching problem.
- Address mode formation happens primarily in combiners (before selection), not in the selector itself.
- Multi-slot pseudo instructions are selected during ISel; concrete slot materialization is deferred to packetization.

## Invariants Downstream Code Depends On
- Zero constants materialized through the selected canonical opcode path.
- Frame-index nodes that match addressing patterns must be converted to target frame-index operands.
- Nodes marked machine opcode must not be re-selected (`NodeId = -1` path).
- Post-increment and multidimensional addressing patterns must be formed before instruction selection (in combiners).
- Selected multi-slot pseudos must have equivalent itineraries across all materialization alternatives.
