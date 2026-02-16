# AIE Backend Build System and Module Dependencies

Scope: `llvm/lib/Target/AIE/**/CMakeLists.txt` on branch `aie-public`.

Files analyzed:
- `llvm/lib/Target/AIE/CMakeLists.txt`
- `llvm/lib/Target/AIE/MCTargetDesc/CMakeLists.txt`
- `llvm/lib/Target/AIE/AsmParser/CMakeLists.txt`
- `llvm/lib/Target/AIE/Disassembler/CMakeLists.txt`
- `llvm/lib/Target/AIE/InstPrinter/CMakeLists.txt`
- `llvm/lib/Target/AIE/TargetInfo/CMakeLists.txt`
- `llvm/lib/Target/AIE/Utils/CMakeLists.txt`

## 1. Library Targets Defined

AIE defines one target-level codegen target plus several component libraries:

- `AIECodeGen` target declared with `add_llvm_target(...)` (produces `LLVMAIECodeGen` build artifact naming in LLVM conventions)
- `LLVMAIEDesc` (`MCTargetDesc/CMakeLists.txt`)
- `LLVMAIEAsmParser` (`AsmParser/CMakeLists.txt`)
- `LLVMAIEDisassembler` (`Disassembler/CMakeLists.txt`)
- `LLVMAIEAsmPrinter` (`InstPrinter/CMakeLists.txt`)
- `LLVMAIEInfo` (`TargetInfo/CMakeLists.txt`)
- `LLVMAIEUtils` (`Utils/CMakeLists.txt`)

All are added to component group `AIE`.

Name mapping (declaration form -> conventional produced library name):
- `add_llvm_target(AIECodeGen)` -> `LLVMAIECodeGen`
- `add_llvm_component_library(LLVMAIEDesc)` -> `LLVMAIEDesc`
- `add_llvm_component_library(LLVMAIEAsmParser)` -> `LLVMAIEAsmParser`
- `add_llvm_component_library(LLVMAIEDisassembler)` -> `LLVMAIEDisassembler`
- `add_llvm_component_library(LLVMAIEAsmPrinter)` -> `LLVMAIEAsmPrinter`
- `add_llvm_component_library(LLVMAIEInfo)` -> `LLVMAIEInfo`
- `add_llvm_component_library(LLVMAIEUtils)` -> `LLVMAIEUtils`

## 2. Dependencies Between Libraries (`LINK_COMPONENTS`)

Note: `LINK_COMPONENTS` entries use LLVM component names (for example `AIECodeGen`, `AIEDesc`) rather than always spelling the full `LLVM...` artifact name.

### `AIECodeGen`
Depends on:
- `Analysis`
- `AsmPrinter`
- `Core`
- `CodeGen`
- `CodeGenTypes`
- `IPO`
- `MC`
- `AIEAsmPrinter`
- `AIEDesc`
- `AIEInfo`
- `AIEUtils`
- `Scalar`
- `SelectionDAG`
- `Support`
- `Target`
- `TargetParser`
- `TransformUtils`
- `GlobalISel`
- `Vectorize`

### `LLVMAIEDesc`
Depends on:
- `MC`
- `AIEInfo`
- `AIEAsmPrinter`
- `Support`
- `TargetParser`

### `LLVMAIEAsmParser`
Depends on:
- `MC`
- `MCParser`
- `AIEDesc`
- `AIEInfo`
- `AIEUtils`
- `Support`
- `TargetParser`

### `LLVMAIEDisassembler`
Depends on:
- `MCDisassembler`
- `AIECodeGen`
- `AIEInfo`
- `MC`
- `Support`

### `LLVMAIEAsmPrinter`
Depends on:
- `MC`
- `AIEUtils`
- `Support`

### `LLVMAIEInfo`
Depends on:
- `Support`
- `MC`

### `LLVMAIEUtils`
Depends on:
- `Analysis`
- `CodeGen`
- `Core`
- `Support`
- `TransformUtils`

## 3. TableGen Invocations and Outputs

TableGen is orchestrated from `llvm/lib/Target/AIE/CMakeLists.txt`.

### Shared AIE1/AIE base definitions (`LLVM_TARGET_DEFINITIONS aie1/AIE1.td`)
Generated files:
- `AIEGenAsmMatcher.inc` (`-gen-asm-matcher`)
- `AIEGenAsmWriter.inc` (`-gen-asm-writer`)
- `AIEGenCallingConv.inc` (`-gen-callingconv`)
- `AIEGenDAGISel.inc` (`-gen-dag-isel`)
- `AIEGenDisassemblerTables.inc` (`-gen-disassembler`)
- `AIEGenGlobalISel.inc` (`-gen-global-isel`)
- `AIEGenFormats.inc` (`-gen-instr-format`)
- `AIEGenInstrInfo.inc` (`-gen-instr-info`, base class `AIEBaseInstrInfo`)
- `AIEGenMCCodeEmitter.inc` (`-gen-emitter`)
- `AIEGenMCPseudoLowering.inc` (`-gen-pseudo-lowering`)
- `AIEGenRegisterBank.inc` (`-gen-register-bank`, base class `AIEBaseRegisterBankInfo`)
- `AIEGenRegisterInfo.inc` (`-gen-register-info`, base class `AIEBaseRegisterInfo`)
- `AIEGenSubtargetInfo.inc` (`-gen-subtarget`)

Commented-out generators exist for compressed/system operands.

### AIE2 definitions (`LLVM_TARGET_DEFINITIONS AIE2.td`)
Generated files include:
- asm matcher/writer, calling conv, disassembler, formats, instr info, emitter, pseudo lowering,
- register bank/info, subtarget info,
- GlobalISel,
- custom AIE generators:
  - `AIE2GenMemoryCycles.inc` (`-gen-aie-memory-cycles`)
  - `AIE2GenPreSchedLowering.inc` (`-gen-aie-presched-lowering`)
  - `AIE2GenSplitInstrTables.inc` (`-gen-aie-split-instr-tables`)
  - `AIE2GenVarInstructionItin.inc` (`-gen-aie-alternate-itinerary-emitter`)
- GISel combiner tables:
  - `AIE2GenPreLegalizerGICombiner.inc`
  - `AIE2GenPostLegalizerGIGenericCombiner.inc`
  - `AIE2GenPostLegalizerGICustomCombiner.inc`

### AIE2P definitions (`LLVM_TARGET_DEFINITIONS AIE2P.td`)
Generated files include:
- instr info, register info, calling conv, subtarget info,
- memory cycles, pre-sched lowering, split-instr tables,
- register bank, GlobalISel,
- asm writer, formats, emitter, disassembler,
- GISel combiner tables:
  - `AIE2PGenPreLegalizerGICombiner.inc`
  - `AIE2PGenPostLegalizerGIGenericCombiner.inc`
  - `AIE2PGenPostLegalizerGICustomCombiner.inc`
- asm matcher,
- alternate itinerary emitter (`AIE2PGenVarInstructionItin.inc`).

Clarification:
- The current AIE2P TableGen block does **not** emit SelectionDAG matcher tables
  (`-gen-dag-isel`) or MC pseudo-lowering tables (`-gen-pseudo-lowering`).
  This aligns with the modern GlobalISel-centric AIE2P codegen path.

Public tablegen target:
- `add_public_tablegen_target(AIECommonTableGen)`

## 4. Source File Lists Per Target

## `AIECodeGen` (from `llvm/lib/Target/AIE/CMakeLists.txt`)

### Common/base sources
- `AIEAddressSpaceFlattening.cpp`
- `AIEBaseAliasAnalysis.cpp`
- `AIEBaseAsmPrinter.cpp`
- `AIEBaseFrameLowering.cpp`
- `AIEBaseHardwareLoops.cpp`
- `AIEBaseISelLowering.cpp`
- `AIEBaseInstrInfo.cpp`
- `AIEBaseRegisterInfo.cpp`
- `AIEBasePipelinerLoopInfo.cpp`
- `AIEBaseInstructionSelector.cpp`
- `AIEBaseTargetTransformInfo.cpp`
- `AIEClusterBaseAddress.cpp`
- `AIECombinerHelper.cpp`
- `AIEBaseRegisterBankInfo.cpp`
- `AIEBaseSubtarget.cpp`
- `AIEBaseTargetMachine.cpp`
- `AIECallLowering.cpp`
- `AIEDataDependenceHelper.cpp`
- `AIEDumpArtifacts.cpp`
- `AIEEliminateDuplicatePHI.cpp`
- `AIEFinalizeBundle.cpp`
- `AIEGlobalCombiner.cpp`
- `AIEGlobalCombinerPtrMods.cpp`
- `AIEHazardRecognizer.cpp`
- `AIEInterBlockScheduling.cpp`
- `AIEISelDAGToDAG.cpp`
- `AIELegalizerHelper.cpp`
- `AIELiveRegs.cpp`
- `AIELoopClass.cpp`
- `AIEMachineAlignment.cpp`
- `AIEMachineFunctionInfo.cpp`
- `AIEMachineScheduler.cpp`
- `AIEMaxLatencyFinder.cpp`
- `AIEMCInstLower.cpp`
- `AIEMIRFormatter.cpp`
- `AIEMultiSlotInstrMaterializer.cpp`
- `AIEPostPipeliner.cpp`
- `AIEPostSelectOptimize.cpp`
- `AIEPseudoBranchExpansion.cpp`
- `AIEPtrModOptimizer.cpp`
- `AIERegClassConstrainer.cpp`
- `AIERegMemEventTracker.cpp`
- `AIESlotCounts.cpp`
- `AIESpillSlotOptimization.cpp`
- `AIESlotStatistics.cpp`
- `AIESlotUtils.cpp`
- `AIESplitInstructionRewriter.cpp`
- `AIESubRegConstrainer.cpp`
- `AIESWPSolver.cpp`
- `AIESuperRegRewriter.cpp`
- `AIESuperRegUtils.cpp`
- `AIETargetObjectFile.cpp`
- `AIE2AsmPrinter.cpp`
- `AIE2FrameLowering.cpp`
- `AIE2InstrInfo.cpp`
- `AIE2InstructionSelector.cpp`
- `AIE2ISelLowering.cpp`
- `AIE2LegalizerInfo.cpp`
- `AIE2PostLegalizerCustomCombiner.cpp`
- `AIE2PostLegalizerGenericCombiner.cpp`
- `AIE2PreLegalizerCombiner.cpp`
- `AIE2RegisterBankInfo.cpp`
- `AIE2RegisterInfo.cpp`
- `AIE2Subtarget.cpp`
- `AIE2TargetMachine.cpp`
- `AIE2TargetTransformInfo.cpp`
- `AIETiedRegOperands.cpp`
- `AIEUnallocatedSuperRegRewriter.cpp`
- `ReservedRegsLICM.cpp`
- `AIEOutlineMemoryGEP.cpp`
- `AIEWawRegRewriter.cpp`

### AIE1 sources
- `aie1/AIE1AsmPrinter.cpp`
- `aie1/AIE1AsmPrinter.cpp` (listed twice in CMake)
- `aie1/AIE1DelaySlotFiller.cpp`
- `aie1/AIE1FrameLowering.cpp`
- `aie1/AIEHazardRecognizerPRAS.cpp`
- `aie1/AIE1InstrInfo.cpp`
- `aie1/AIE1InstructionSelector.cpp`
- `aie1/AIE1ISelLowering.cpp`
- `aie1/AIE1LegalizerInfo.cpp`
- `aie1/AIE1MachineBlockPlacement.cpp`
- `aie1/AIE1RegisterBankInfo.cpp`
- `aie1/AIE1RegisterInfo.cpp`
- `aie1/AIE1Subtarget.cpp`
- `aie1/AIE1TargetMachine.cpp`

### AIE2P sources
- `aie2p/AIE2PSubtarget.cpp`
- `aie2p/AIE2PTargetMachine.cpp`
- `aie2p/AIE2PTargetTransformInfo.cpp`
- `aie2p/AIE2PRegisterInfo.cpp`
- `aie2p/AIE2PLegalizerInfo.cpp`
- `aie2p/AIE2PInstrInfo.cpp`
- `aie2p/AIE2PFrameLowering.cpp`
- `aie2p/AIE2PRegisterBankInfo.cpp`
- `aie2p/AIE2PISelLowering.cpp`
- `aie2p/AIE2PInstructionSelector.cpp`
- `aie2p/AIE2PPostLegalizerCustomCombiner.cpp`
- `aie2p/AIE2PPostLegalizerGenericCombiner.cpp`
- `aie2p/AIE2PPreLegalizerCombiner.cpp`

## `LLVMAIEDesc`
- `AIE1AsmBackend.cpp`
- `AIE2AsmBackend.cpp`
- `AIEBaseAsmBackend.cpp`
- `AIEBaseMCFormats.cpp`
- `AIEBaseMCCodeEmitter.cpp`
- `AIEELFObjectWriter.cpp`
- `AIEMCAsmInfo.cpp`
- `AIEMCCodeEmitter.cpp`
- `AIEMCExpr.cpp`
- `AIEMCFixupKinds.cpp`
- `AIE1MCFixupKinds.cpp`
- `AIEMCFormats.cpp`
- `AIEMCInstrInfo.cpp`
- `AIEMCTargetDesc.cpp`
- `AIETargetAsmStreamer.cpp`
- `AIETargetELFStreamer.cpp`
- `AIE2MCFixupKinds.cpp`
- `AIE2MCFormats.cpp`
- `AIE2MCTargetDesc.cpp`
- `AIE2MCCodeEmitter.cpp`
- `AIEFormat.cpp`
- `aie2p/AIE2PMCTargetDesc.cpp`
- `aie2p/AIE2PMCFormats.cpp`
- `aie2p/AIE2PAsmBackend.cpp`
- `aie2p/AIE2PMCFixupKinds.cpp`
- `aie2p/AIE2PMCCodeEmitter.cpp`

## `LLVMAIEAsmParser`
- `AIE1AsmParser.cpp`
- `AIE2AsmParser.cpp`
- `AIE2PAsmParser.cpp`
- `AIEBaseOperand.cpp`

## `LLVMAIEDisassembler`
- `AIEDisassembler.cpp`
- `AIE2Disassembler.cpp`
- `AIE2PDisassembler.cpp`
- `InitializeDisassemblers.cpp`

## `LLVMAIEAsmPrinter`
- `AIECommonInstPrinter.cpp`
- `AIEInstPrinter.cpp`
- `AIE2InstPrinter.cpp`
- `AIE2PInstPrinter.cpp`

## `LLVMAIEInfo`
- `AIETargetInfo.cpp`

## `LLVMAIEUtils`
- `AIEBaseInfo.cpp`
- `AIEIRUtils.cpp`
- `AIELoopUtils.cpp`
- `AIEMachineBasicBlockUtils.cpp`

## 5. Build Ordering Constraints

Ordering is implied by both subdirectory layout and link dependencies.

Top-level AIE build sequence in `llvm/lib/Target/AIE/CMakeLists.txt`:
1. run TableGen for AIE/AIE2/AIE2P (`*.inc` generation),
2. define `AIECodeGen`,
3. add subdirectories:
   - `AsmParser`
   - `Disassembler`
   - `InstPrinter`
   - `MCTargetDesc`
   - `TargetInfo`
   - `Utils`

Practical dependency constraints:
- `AIECodeGen` requires `AIEAsmPrinter`, `AIEDesc`, `AIEInfo`, `AIEUtils`.
- `LLVMAIEDesc` requires `AIEInfo` and `AIEAsmPrinter`.
- `LLVMAIEAsmParser` requires `AIEDesc`, `AIEInfo`, `AIEUtils`.
- `LLVMAIEDisassembler` requires `AIECodeGen` and `AIEInfo`.

Thus, even if subdirectory declaration order is not strictly topological, CMake/LLVM target dependency resolution still enforces module-level build completion before dependents are linked.

### Explicit module dependency edges

- `LLVMAIEInfo` -> (`MC`, `Support`) only, no AIE-internal prerequisites.
- `LLVMAIEUtils` -> (`Analysis`, `CodeGen`, `Core`, `Support`, `TransformUtils`) only, no AIE-internal prerequisites.
- `LLVMAIEAsmPrinter` -> `LLVMAIEUtils`
- `LLVMAIEDesc` -> `LLVMAIEInfo`, `LLVMAIEAsmPrinter`
- `AIECodeGen` -> `LLVMAIEAsmPrinter`, `LLVMAIEDesc`, `LLVMAIEInfo`, `LLVMAIEUtils`
- `LLVMAIEAsmParser` -> `LLVMAIEDesc`, `LLVMAIEInfo`, `LLVMAIEUtils`
- `LLVMAIEDisassembler` -> `AIECodeGen`, `LLVMAIEInfo`

### Topological implementation/build order (module level)

One valid dependency-ordered sequence is:
1. `LLVMAIEInfo`
2. `LLVMAIEUtils`
3. `LLVMAIEAsmPrinter`
4. `LLVMAIEDesc`
5. `AIECodeGen`
6. `LLVMAIEAsmParser`
7. `LLVMAIEDisassembler`

This ordering is useful for both:
- incremental backend bring-up planning, and
- validating architecture-level dependency claims in `09-architecture/architecture.md`.

## 6. Custom Build Rules or Configuration

Notable customizations:
- Custom TableGen backends beyond upstream defaults:
  - `-gen-aie-memory-cycles`
  - `-gen-aie-presched-lowering`
  - `-gen-aie-split-instr-tables`
  - `-gen-aie-alternate-itinerary-emitter`
- GlobalISel combiner generation with explicit combiner names for AIE2/AIE2P.
- Common public TableGen aggregation target: `AIECommonTableGen`.
- Component grouping via `add_llvm_component_group(AIE)` and `ADD_TO_COMPONENT AIE`.

Observations relevant to dependency analysis:
- `aie1/AIE1AsmPrinter.cpp` appears twice in `AIECodeGen` source list.
- Some legacy/experimental generators are present but commented out (compressed inst emitter, system operands), indicating optional generation paths not currently active.

## Module Boundary Map (Summary)

- Frontend assembly parsing: `LLVMAIEAsmParser`
- MC/encoding/format descriptors: `LLVMAIEDesc`
- Instruction printing: `LLVMAIEAsmPrinter`
- Disassembly: `LLVMAIEDisassembler`
- Target registration/info: `LLVMAIEInfo`
- Cross-cutting utility layer: `LLVMAIEUtils`
- Core codegen and passes: `AIECodeGen`

This boundary map is the foundation for higher-level architecture, ISA, and scheduling documentation under `docs/aie-spec/`.
