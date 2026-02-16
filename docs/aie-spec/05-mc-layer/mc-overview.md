# AIE MC Layer Overview

Scope analyzed:
- `llvm/lib/Target/AIE/MCTargetDesc/*`
- `llvm/lib/Target/AIE/AsmParser/*`
- `llvm/lib/Target/AIE/Disassembler/*`
- `llvm/lib/Target/AIE/InstPrinter/*`

## 1. MC Layer Architecture

The AIE MC stack follows standard LLVM layering with AIE-specific format/fixup logic:
- Target registration and MC factory wiring: `AIEMCTargetDesc.cpp`
- Object writer and ELF streamer: `AIEELFObjectWriter.cpp`, `AIETargetELFStreamer.cpp`
- Code emission and fixup mapping: `AIEBaseMCCodeEmitter.*`, `AIEMCFixupKinds.*`, arch-specific `*MCFixupKinds.*`
- Assembly parse/print: `AIEBaseAsmParser.h`, `AIE{1,2,2P}AsmParser.cpp`, `AIE{,2,2P}InstPrinter.cpp`, `AIECommonInstPrinter.*`
- Disassembly: `AIE{,2,2P}Disassembler.cpp` plus generated decoder tables.

AIE-specific differentiation is in:
- variable-width VLIW packet formats,
- multi-field reloc/fixup selection,
- composite-instruction fixup translation (sub-instruction scope -> packet scope).

## 2. Object File Format (ELF specifics)

From `AIEELFObjectWriter.cpp` and `AIETargetELFStreamer.cpp`:
- ELF machine type: `EM_AIE`.
- Relocation model: RELA (`HasRelocationAddend = true`).
- ELF `e_flags` encodes architecture variant:
  - `EF_AIE_AIE1` for `Triple::aie`
  - `EF_AIE_AIE2` for `Triple::aie2`
  - `EF_AIE_AIE2P` for `Triple::aie2p`
- Linker-padding control: streamer `finish()` switches to `.text` (`SHT_PROGBITS`) and emits code alignment `Align(16)` so padding is filled with target NOPs instead of zero bytes.

From `AIEMCAsmInfo.cpp`:
- `.bss` emission uses ELF section directives (`UsesELFSectionDirectiveForBSS = true`).
- Data directives are:
  - 16-bit: `\t.short\t`
  - 32-bit: `\t.word\t`

## 3. Relocation Types and Fixups

### 3.1 Relocation mapping policy

`AIEELFObjectWriter::getRelocType()` chooses arch-specific mapping:
- AIE1: `getRelocTypeAIE1`
- AIE2: `getRelocTypeAIE2`
- AIE2P: `getRelocTypeAIE2P`

Target fixups are dense one-to-one maps:
- `fixup_aie_*` -> `R_AIE_*`
- `fixup_aie2_*` -> `R_AIE_*` (index-shifted from `fixup_aie2_0`)
- `fixup_aie2p_*` -> `R_AIE_*` (index-shifted from `fixup_aie2p_0`)

Non-target base LLVM fixup handling includes:
- `FK_Data_4` mapped per architecture to DM word reloc:
  - AIE1: `R_AIE_72`
  - AIE2: `R_AIE_50`
  - AIE2P: `R_AIE_62`

### 3.2 Fixup model used by the emitter

`AIEMCFixupKinds` models fixups as one-or-more bitfields (`FixupField {Offset, Size}`), because AIE relocations can patch split immediates.

Selection of a concrete fixup uses:
1. relocatable operand fields from format metadata,
2. packet/instruction format size,
3. optional signedness disambiguation (`FixupFlag`) via opcode-specific flags.

This is driven by generated fixup tables in:
- `MCTargetDesc/FixupInfo/AIE1FixupInfo.inc`
- `MCTargetDesc/FixupInfo/AIE2FixupInfo.inc`
- `MCTargetDesc/FixupInfo/AIE2PFixupInfo.inc`

## 4. Section Layout and ABI-facing details

Observed ABI-facing MC details:
- Initial CFI state in MC asm info is defined as CFA = `SP` (`AIEMCTargetDesc.cpp`).
- Default backend ABI enum in asm backend: `AIEABI::ABI_VITIS` (`AIEBaseAsmBackend.h`).
- `needsRelocateWithSymbol()` is currently conservative (`true`) for all relocations.
- `AIEELFObjectWriter` notes no PC-relative call relocation handling for AIE1/AIE2 in current logic.

## 5. Binary Interface Contract exposed by MC layer

At object-level, this layer defines:
- machine identity (`EM_AIE`, `EF_AIE_*`),
- relocation namespace and fixup-to-reloc lowering policy,
- packet-width decoding/encoding interpretation (2/4/6/8/10/12/14/16-byte families for AIE2/AIE2P; 2/4/8/12/16-byte families for AIE1),
- NOP fill bytes used for code alignment tails (target-specific in asm backends).

In practice, linker/objdump interoperability depends on these exact choices.
