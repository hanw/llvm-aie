# AIE Backend Optimization Passes

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIE2PreLegalizerCombiner.cpp`
- `llvm/lib/Target/AIE/AIE2PostLegalizerGenericCombiner.cpp`
- `llvm/lib/Target/AIE/AIE2PostLegalizerCustomCombiner.cpp`
- `llvm/lib/Target/AIE/aie2p/AIE2PPreLegalizerCombiner.cpp`
- `llvm/lib/Target/AIE/aie2p/AIE2PPostLegalizerGenericCombiner.cpp`
- `llvm/lib/Target/AIE/aie2p/AIE2PPostLegalizerCustomCombiner.cpp`
- `llvm/lib/Target/AIE/AIECombinerHelper.*`
- `llvm/lib/Target/AIE/AIEPtrModOptimizer.*`
- `llvm/lib/Target/AIE/AIEGlobalCombiner*.{h,cpp}`
- `llvm/lib/Target/AIE/AIEPostSelectOptimize.cpp`
- `llvm/lib/Target/AIE/AIEBaseHardwareLoops.cpp`
- `llvm/lib/Target/AIE/AIEAddressSpaceFlattening.cpp`
- `llvm/lib/Target/AIE/AIEClusterBaseAddress.cpp`
- `llvm/lib/Target/AIE/ReservedRegsLICM.cpp`

## Public Interface (Pass Entry Points)

### Combiner Passes
- `createAIE2PreLegalizerCombiner()`
- `createAIE2PostLegalizerGenericCombiner()`
- `createAIE2PostLegalizerCustomCombiner()`
- `createAIE2PPreLegalizerCombiner()`
- `createAIE2PPostLegalizerGenericCombiner()`
- `createAIE2PPostLegalizerCustomCombiner()`

### Address and Memory Passes
- `createAIEAddressSpaceFlattening()`
- `createAIEClusterBaseAddress()`
- `createAIEPtrModOptimizer()`
- `createAIEOutlineMemoryGEP()`
- `createReservedRegsLICMPass()`

### Post-Selection and RA Passes
- `createAIEPostSelectOptimize()`
- `createAIESplitInstrBuilder()`
- `createAIESplitInstrReplacer()`
- `createAIERegClassConstrainer()`
- `createAIESpillSlotOptimization()`
- `createAIESuperRegRewriter()`
- `createAIEUnallocatedSuperRegRewriter()`
- `createAIEWawRegRewriter()`

### Loop and Control Flow Passes
- `createAIEBaseHardwareLoopsPass()`
- `createAIEPseudoBranchExpansion()`

### Bundle and Alignment Passes
- `createAIEFinalizeBundle()`
- `createAIEMachineAlignment()`

## Optimization Stages in Pipeline

### Stage 1: Address Space Flattening (`AIEAddressSpaceFlattening`)

Runs as the first pre-legalizer pass. Retypes all pointers to default address space 0 and replaces `G_ADDRSPACE_CAST` with `COPY`.

Design rationale: Address spaces carry banking annotations from the frontend (see address space definitions in `target-machine.md`). Flattening them early allows all subsequent MIR passes to operate on a uniform pointer type without needing address-space-aware logic.

### Stage 2: Pre-Legalizer Combiner

Generic-MI canonicalization before the legalizer runs:
- Custom intrinsic combining.
- Shuffle/vector sequence canonicalization.
- AIE2P-specific: intrinsic handling for vector shift zero-combine and extraction legalization shape checks.

### Stage 3: Post-Legalizer Generic Combiner

TableGen-driven and helper-driven generic-MI combines after legalization:
- Standard LLVM generic combines adapted for AIE legal types.
- Runs before address clustering to ensure canonical forms.

### Stage 4: Address Clustering and Pointer-Mod Analysis

#### `AIEClusterBaseAddress`
Chains `G_PTR_ADD` operations to enable post-increment addressing patterns.

Key transformations:
- Identifies load/store sequences accessing consecutive addresses.
- Swaps non-constant/constant offsets in `G_PTR_ADD` chains to maximize constant-offset chains.
- Enables downstream combiner to form post-increment load/store patterns.

Controlled by `-aie-address-chaining` (on by default at O>0).

#### `AIEPtrModOptimizer`
An **analysis pass** that scans basic blocks and records profitable pointer-modifier combine opportunities:
- Uses dominator tree, alias analysis, and custom dependence helper.
- Produces `AIE::FoundCombiners` map keyed by combine roots.
- Results are consumed by the post-legalizer custom combiner.
- Does not modify MIR directly.

Controlled by `-aie-global-ptr-mod-opt` (on by default at O>0).

### Stage 5: Post-Legalizer Custom Combiner

Uses pointer-mod analysis results (if enabled) and custom combine rules:
- Post-increment load/store formation.
- 2D/3D addressing pattern formation.
- Pointer-add + load/store offset folding.
- Shared pointer-add reuse.

Deletion of replaced instructions is **deferred** and finalized after combine iterations (`finalizeDeferredDeletes`) to avoid invalidating raw pointers used by other combine candidates.

### Stage 6: Post-Select Optimizer (`AIEPostSelectOptimize`)

Cleans and canonicalizes selected MIR before RA/scheduling:

| Transformation | Description |
|---------------|-------------|
| Subreg-copy folding | Fold subreg-copy chains into super-reg copies |
| INSERT_SUBREG rewrite | Rewrite eligible chains into `REG_SEQUENCE` |
| Reserved-reg cleanup | Remove redundant reserved-register copies/immediates via SSA copy tracking |
| Address reg duplication | Duplicate addressing registers to reduce downstream conflicts |
| Memory operand fixup | Fix memory operand info for loads using non-pointer operands |
| Store-flush conversion | AIE2P-specific `VST.FLUSH` + conversion chain adjustment |

Pass invariants:
- Requires selected MIR; skips failed-isel functions.
- Preserves CFG.

### Stage 7: Reserved Registers LICM (`ReservedRegsLICM`)

Loop-invariant code motion specifically for **reserved physical registers**:
- Uses `LivePhysRegs` analysis to identify loop-invariant definitions of reserved registers.
- Hoists qualifying definitions out of loops.
- Particularly useful for control/status register operations that are invariant within loop bodies.

### Stage 8: Pre-RA Optimization

#### Software Pipeliner (MachinePipeliner)
- Enabled at O2+ via `-aie-enable-pipeliner`.
- Uses target-specific SMS mutations from `AIEBaseSubtarget::getSMSMutations()`.
- Followed by DCE to clean up dead code from pipeliner transformations.

#### SubReg Constrainer (`AIESubRegConstrainer`)
- Runs before PHI elimination.
- Constrains register classes for subregister operands.

#### Memory GEP Outlining (`AIEOutlineMemoryGEP`)
- Optional pass in ISel preparation.
- Outlines common memory address computation patterns.

#### Early If-Conversion (`EarlyIfConverter`)
- Available via `addILPOpts()`.
- Converts simple if-then-else patterns to predicated execution.

### Stage 9: Staged Register Allocation

The RA flow is decomposed by register family for the deeply partitioned register file:

1. Optional coalescer rerun.
2. Optional split-instr builder (creates RA-friendly instruction forms).
3. Regclass constrainer.
4. **Selective greedy allocation phases** — allocate by register family:
   - Phase 1: Modifier registers (M).
   - Phase 2: 3D dimension registers.
   - Phase 3: 3D + 2D dimension registers.
5. Super-reg rewriter between phases.
6. Optional WAW reg rewriter + extra RA pass.
7. Final virtual register rewriter.
8. AIE2P: optional unallocated-superreg rewriter in fine-grained mode.

### Stage 10: Post-RA Passes

#### Spill Slot Optimization (`AIESpillSlotOptimization`)
- Minimizes stack usage by reusing spill slots.
- Controlled by `-aie-stack-minimize` (on by default).

#### Split-Instruction Replacer (`AIESplitInstrReplacer`)
- Paired with split-instr builder from pre-RA.
- Replaces split instruction forms with final forms after allocation.

#### Hardware Loop Lowering (`AIEBaseHardwareLoops`)
- Finds low-overhead loop pseudo patterns and expands them to concrete `loop_start`/`loop_end` behavior.
- Recognizes loop pseudo instruction families through `AIEBaseInstrInfo` hooks (`isHardwareLoop*`).
- Processes nested loops recursively.
- Runs after RA, before post-RA scheduling.

#### Pseudo Branch Expansion (`AIEPseudoBranchExpansion`)
- Expands pseudo branch instructions to concrete branch forms.
- Runs before post-RA scheduling.

#### Post-RA Machine Scheduler
- **Mandatory for correctness** — the exposed pipeline architecture requires compiler-generated hazard-free schedules.
- Inserts NOPs as needed for pipeline hazard avoidance.
- Uses target-specific scheduling mutations and resource models.

#### Bundle Finalization (`AIEFinalizeBundle`)
- Assembles scheduled instructions into VLIW bundles.
- Selects minimal composite format that covers occupied slots.
- Fills unused slots with per-slot NOPs.

#### Machine Alignment (`AIEMachineAlignment`)
- Final pass ensuring instruction alignment meets hardware requirements.

## Custom Combiner Themes (`AIECombinerHelper`)

Main combine categories:

| Category | Patterns |
|----------|----------|
| Address generation | Pointer-add + load/store offset folding, post-increment formation, shared pointer-add reuse |
| Vector/shuffle | Broadcast, extract+broadcast, splat, pad/unpad, concat/unmerge |
| Global value | Global value + offset simplification |
| Load/store | Splitting, offset normalization |
| Accumulator | Accumulator exposure patterns |
| Memset | Memset-oriented transforms |

Design note: There is a clear split between generated match-table rules and hand-coded helper rules for target idioms not easily expressed in pure TableGen.

## AIE2 vs AIE2P Differences
- Both use pre/post legalizer generic/custom combiner triad.
- AIE2P pre-legalizer includes explicit intrinsic handling for vector shift zero-combine and extraction legalization shape checks.
- AIE2P staged RA path optionally adds `AIEUnallocatedSuperRegRewriter` in fine-grained mode.
- AIE2P emergency spill slots are managed by frame lowering (see `frame-lowering.md`).

## Key Design Decisions
- Multi-phase combining is intentional: canonicalize first, then legalize, then exploit legal forms plus address analyses.
- Pointer-mod combining is analysis-driven with deferred deletion to maintain multi-candidate safety.
- Post-select pass acts as a practical peephole layer to reduce reserved-register noise and improve machine-level canonical form before heavy RA/scheduling.
- Hardware loop lowering is separated from earlier loop formation to keep final scheduling/legal packet formation coherent.
- AIE optimization strategy intentionally mixes classic "combiner" passes with late RA-aware rewrite passes, because many profitable transforms are only safe/meaningful after register classes and bundle constraints are visible.
- Address space flattening is deliberately early — it simplifies all downstream passes by removing address-space polymorphism.
- Post-RA scheduling is not optional — it is a correctness requirement for the exposed pipeline architecture.

## Invariants Downstream Code Depends On
- Custom post-legalizer combiners may expect `FoundCombiners` availability when pointer-mod optimization is enabled.
- Deferred-deletion protocol must be respected to avoid use-after-free during combine iterations.
- Post-select cleanup assumes SSA form and selected MIR.
- Hardware loop pseudo patterns must be in recognizable form when entering hardware-loop expansion pass.
- Split-instr builder/replacer, when enabled, must remain paired and ordered correctly; downstream RA and scheduling assume the transformed instruction set shape.
- Address space flattening must complete before legalizer runs.
- Post-RA scheduler must run before bundle finalization.
