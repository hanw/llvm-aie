# AIE ISA Overview (TableGen-Driven)

This document summarizes the ISA structure encoded in all 48 TableGen files under `llvm/lib/Target/AIE`.

Coverage note:
- All `.td` files under `llvm/lib/Target/AIE` were reviewed (48 total), including `aie1/` legacy definitions.
- Primary synthesis target is `aie2`/`aie2p`, because those are the modern backend paths emphasized by this documentation set.

TableGen coverage map (all included in this analysis):
- Target roots: `AIE2.td`, `AIE2P.td`, `aie1/AIE1.td`
- Registers/banks/operands: `*RegisterInfo*.td`, `*RegisterBanks*.td`, `AIE2RegOperandDef.td`, `AIE2PRegOperandDef.td`
- Instructions/formats/composites: `*InstrInfo*.td`, `*InstrFormats*.td`, `*CompositeFormats*.td`, `AIE2PCompositeFormatsInclude.td`, `*Slots*.td`, `*MultiSlotPseudoInstrInfo*.td`
- Calling convention: `AIE2CallingConv.td`, `aie2p/AIE2PCallingConv.td`, `aie1/AIE1CallingConv.td`
- Scheduling: `AIE2Schedule.td`, `aie2p/AIE2PSchedule.td`, `aie2p/AIE2PGenSchedule.td`, `aie1/AIE1Schedule.td`
- Patterns/GISel/combines: `AIEBaseInstrPatterns.td`, `AIE2InstrPatterns.td`, `aie2p/AIE2PInstrPatterns.td`, `AIEInstrGISel.td`, `AIECombine.td`
- Generated-support definitions: `AIE2GenInstrFormats.td`, `AIE2GenInstrInfo.td`, `AIE2GenFixupInstrInfo.td`, `AIE2GenRegisterInfo.td`, `aie2p/AIE2PGenInstrInfo.td`
- Immediate definitions: `aie2p/AIE2PImmOperands.td`

## Scope and Targets

Primary target roots:
- `llvm/lib/Target/AIE/AIE2.td` — includes `AIE2RegisterInfo.td`, `AIE2Schedule.td`, `AIE2CallingConv.td`, `AIE2RegisterBanks.td`, `AIEInstrGISel.td`, `AIE2InstrInfo.td`, `AIECombine.td`
- `llvm/lib/Target/AIE/AIE2P.td` — includes `aie2p/AIE2PRegisterInfo.td`, `aie2p/AIE2PSchedule.td`, `aie2p/AIE2PCallingConv.td`, `aie2p/AIE2PRegisterBanks.td`, `AIEInstrGISel.td`, `aie2p/AIE2PInstrInfo.td`, `AIECombine.td`

Processor model entries:
- `ProcessorModel<"aie2", AIE2SchedModel, []>` in `AIE2.td`
- `ProcessorModel<"aie2p", AIE2PSchedModel, []>` in `AIE2P.td`

Interpretation:
- ISA variants are modeled as separate targets (`AIE2`, `AIE2P`) with shared base infrastructure (`AIEBase*`) and per-variant register/schedule/instruction descriptions.
- From backend perspective, architecture names are `aie2` and `aie2p` (used as CPU/arch selectors and as the architecture component of toolchain triples).

Target triple clarification:
- The TableGen layer defines processor models (`aie2`, `aie2p`) rather than full OS/ABI triple strings.
- In practice, backend-facing triple support is keyed by these architecture names, with OS/vendor/environment coming from toolchain configuration.
- Typical toolchain triples are `aie2-none-unknown-elf` and `aie2p-none-unknown-elf`.

## VLIW Slot Structure

Both variants use explicit slot-typed instruction classes (`InstSlot`) and then build composites from those slots.

### Design Philosophy (from `AIE2Slots.td` comments)

The VLIW encoding uses a **hierarchical decomposition** where complex VLIW instructions are defined in terms of simpler per-slot instructions at the MC level. This avoids describing the same instruction multiple times and prevents the cross-product explosion of instruction specifications that would result from a flat encoding model.

### AIE2 Slots (`AIE2Slots.td`)

| Slot | Field Name | Bit Width | Default Size | Purpose |
|------|-----------|-----------|-------------|---------|
| `lda_slot` | `lda` | 21 | 4 bytes | Load address path A |
| `ldb_slot` | `ldb` | 16 | 4 bytes | Load address path B |
| `alu_slot` | `alu` | 20 | 4 bytes | Scalar ALU |
| `mv_slot` | `mv` | 22 | 4 bytes | Move/misc |
| `st_slot` | `st` | 21 | 4 bytes | Store |
| `vec_slot` | `vec` | 26 | 4 bytes | Vector operations |
| `lng_slot` | `lng` | 42 | 6 bytes | Long instructions |
| `nop_slot` | `nop` | 1 | 2 bytes | Artificial disambiguation slot |

### AIE2P Slots (`aie2p/AIE2PSlots.td` + `AIE2PSlotInclude.td`)

| Slot | Field Name | Bit Width | Default Size | Purpose |
|------|-----------|-----------|-------------|---------|
| `lda_slot` | `lda` | 20 | 4 bytes | Load address path A |
| `ldb_slot` | `ldb` | 17 | 4 bytes | Load address path B |
| `alu_slot` | `alu` | 20 | 4 bytes | Scalar ALU |
| `mv_slot` | `mv` | 22 | 4 bytes | Move/misc |
| `st_slot` | `st` | 20 | 4 bytes | Store |
| `vec_slot` | `vec` | 26 | 4 bytes | Vector operations |
| `lng_slot` | `lng` | 42 | 6 bytes | Long instructions |
| `nop_slot` | `nop` | 1 | 2 bytes | Artificial disambiguation (via include) |

### Nop Slot Design

The artificial `nop_slot` is used for disambiguation of equivalent formats with different sizes. When an instruction fits in both a smaller and larger format, the encoder uses the smallest composite format that covers the specified slots. The nop slot forces selection of a larger format when needed for alignment or bundle-width requirements.

### Slot Assignment Mechanics

- Instructions are first defined per functional slot (e.g., `AIE2_inst_alu_instr32`, `AIE2P_inst_vec_instr32`).
- Each slot class binds `let Slot = ...` and `DecoderNamespace = slot.SlotName`.
- Final encodings are selected by matching minimal composite formats that cover occupied slots.

## Instruction Encoding Formats

Composite format families are explicit classes in:
- `AIE2CompositeFormats.td` (58 concrete formats for AIE2)
- `aie2p/AIE2PCompositeFormats.td` (AIE2P equivalents)
- `AIE2PCompositeFormatsInclude.td` (AIE2P 16b nop)

### Supported Bundle Widths

| Width | AIE2 Tag Bits | AIE2P Tag Bits |
|-------|-------------|---------------|
| 128-bit | `0b0` (1-bit) | `0b0` (1-bit) |
| 112-bit | `0b1111` (4-bit) | `0b1111` (4-bit) |
| 96-bit | `0b0111` (4-bit) | `0b0111` (4-bit) |
| 80-bit | `0b1011` (4-bit) | `0b1011` (4-bit) |
| 64-bit | `0b0011` (4-bit) | `0b0011` (4-bit) |
| 48-bit | `0b101` (3-bit) | `0b101` (3-bit) |
| 32-bit | `0b1001` (4-bit) | `0b1001` (4-bit) |
| 16-bit | `0b0001` (4-bit) | `0b0001` (4-bit) |

The encoding pattern is: `Inst = {payload_bits, tag_bits}`, where tag bits occupy the least significant positions and uniquely identify the bundle width.

### Representative Composite Definitions

AIE2:
- `I48_LDA_ALU`, `I64_LDA_LDB_ST`, `I96_LDA_LDB_ALU_ST`, `I128_LDB_LDA_ST_ALU_MV_VEC`

AIE2P:
- `I48_LDA_ALU`, `I64_ST_VEC`, `I96_LDA_LDB_ALU_VEC`, `I128_LDA_LDB_ST_ALU_MV_VEC`

### Format Count Summary (AIE2)

| Width | Count | Example Slot Combinations |
|-------|-------|--------------------------|
| 128-bit | 2 | `ldb+lda+st+lng+vec`, `ldb+lda+st+alu+mv+vec` |
| 112-bit | 8 | Various 5-slot combinations |
| 96-bit | 20 | Various 4-slot combinations |
| 80-bit | 18 | Various 3-slot combinations |
| 64-bit | 12 | Various 2-slot combinations |
| 48-bit | 9 | Various 2-slot combinations |
| 32-bit | 6 | Single-slot: `vec`, `st`, `mv`, `ldb`, `lda`, `alu` |
| 16-bit | 1 | `nop` only |

## Register File Summary

TableGen models a strongly partitioned register architecture, not a flat RISC register file.

### Major Scalar/Address Classes

| Domain | AIE2 | AIE2P | Description |
|--------|------|-------|-------------|
| GPR | `eR` (r0–r31, 32-bit) | `eR` (r0–r31, 32-bit) | Primary scalar registers |
| Pair/scalar64 | `eL` (l0–l7, 64-bit) | `eL` (l0–l15, 64-bit) | Paired GPR registers |
| Pointer | `eP` (p0–p7, 20-bit) | `eP` (p0–p7, 20-bit) | Address pointers |
| Modifier | `eM` (m0–m7, 20-bit) | `eM` (m0–m7, 20-bit) | Address modifiers |
| 2D DMA | `eD` (d0–d7, 80-bit) | `eD` (d0–d7, 80-bit) | 2D dimension config (mod+size+stride+count) |
| 3D DMA | `eDS` (d0_3d–d3_3d, 160-bit) | `eDS` (160-bit) | 3D dimension config (pairs of 2D) |
| DMA fields | `eDJ`, `eDN`, `eDC` (20-bit each) | `eDJ`, `eDN`, `eDC` (20-bit each) | Jump/next/count components |
| Special | `sp`, `lr`, `ls`, `le`, `lc`, `CORE_ID` | `sp`, `lr`, `ls`, `le`, `lc`, `CORE_ID` | Stack pointer, link, loop control |

### Major Vector/Accumulator Classes

| Domain | AIE2 | AIE2P |
|--------|------|-------|
| Vector 128-bit | `VEC128` (Q classes) | `VEC128` |
| Vector 256-bit | `VEC256` (W classes, 24 regs) | `VEC256` (W classes) |
| Vector 512-bit | `VEC512` (X classes, 12 regs) | `VEC512` (X classes) |
| Vector 1024-bit | `VEC1024` (Y, 4 regs) | `VEC1024` (Y) |
| Sparse vector | `SPARSEVEC640` (QX, 4 regs) | — |
| Accumulator 256-bit | `ACC256` (AM, 36 regs) | — |
| Accumulator 512-bit | `ACC512` (BM, 18 regs) | `ACC512` (BM) |
| Accumulator 1024-bit | `ACC1024` (CM, 9 regs) | `ACC1024` (CM) |
| Accumulator 2048-bit | — | `ACC2048` (DM, 5 regs) |
| BFP 288-bit | — | `VEC288` (exp+vec composite) |
| BFP 576-bit | — | `VEC576` (EX classes) |
| BFP 1152-bit | — | `VEC1152` (EY classes) |
| Exponent 64-bit | — | `EXPVEC64` (e0–e11) |
| FIFO 512-bit | — | `FIFO512` (load/store FIFO) |
| FIFO 1024-bit | — | `FIFO1024` (combined FIFO) |
| Mask 128-bit | `VEC128` (Q, 4 regs) | `VEC128` (Q, 4 regs) |

### Register Banks (`*RegisterBanks.td`)

| Bank | Base | AIE2 Addition | AIE2P Addition |
|------|------|--------------|----------------|
| `PTRRegBank` | `[eP]` | — | — |
| `MODRegBank` | `[mDm]` | — | — |
| `VRegBank` | `[VEC128..VEC1024]` | — | — |
| `GPRRegBank` | — | `[eR, eL]` | `[eR, eL, eE, EXPVEC64]` |
| `AccRegBank` | — | `[ACC256..ACC1024]` | `[ACC512..ACC2048]` |
| `FifoRegBank` | — | — | `[FIFO512, FIFO1024]` |

### Control and Status Registers

AIE2 defines explicit control registers (`mCRm`, 12 members: `crSat`, `crRnd`, `crFPMask`, `crSRSSign`, etc.) and status registers (`mSRm`, 10 members: `srCarry`, `srSRS_of`, `srFPFlags`, etc.) that are architecturally visible and participate in instruction side effects.

## What Makes This ISA Different from a Standard RISC Target

1. **Slot-first VLIW encoding model.**
   Scheduling/encoding is about legal slot combinations, not single fixed-width opcodes. The hierarchical decomposition (slot → composite) avoids cross-product explosion.

2. **Composite format hierarchy with variable bundle width.**
   Final instruction width (16–128 bits) depends on occupied slot set and format tags. A tag-based format identifier in the least significant bits drives the decoder.

3. **Deeply typed register ecosystem.**
   Scalar, pointer, modifier, vector, accumulator, mask, sparse/BFP/FIFO classes are explicit and frequently non-interchangeable. Operand-specific register subsets (e.g., `mLdaScl`, `mSclSt`, `mMvSclDst`) constrain which registers can appear in which instruction operand positions.

4. **Address generation is architectural.**
   1D/2D/3D post-increment and tied subregister semantics are first-class in instruction forms and register modeling. The 80-bit `eD` and 160-bit `eDS` classes compose modifier+size+stride+count fields as explicit subregisters.

5. **Exposed pipeline + hazard-aware itineraries.**
   Functional-port/resource conflicts and special hazard handling (locks/semaphores/part-word store behavior) are encoded directly in schedule TableGen. The ISA comment states: "this file must model the processor pipeline with reasonably high fidelity in order to enable the generation of correct code."

6. **Heavy generated specialization.**
   AIE2P uses generated instruction/schedule tables with itinerary variants keyed by operand register classes (4010 itinerary classes vs. 278 for AIE2).

7. **Multi-slot pseudo abstraction.**
   A `MultiSlot_Pseudo` mechanism allows codegen to defer slot choice until packetization, while maintaining the invariant that all concrete alternatives have equivalent scheduling behavior.

8. **Block floating-point and FIFO data paths (AIE2P).**
   AIE2P introduces dedicated register classes and data paths for block floating-point operations (`VEC288`/`VEC576`/`VEC1152` with exponent subregisters) and hardware FIFO management (`FIFO512`/`FIFO1024`), reflecting deep learning accelerator specialization.
