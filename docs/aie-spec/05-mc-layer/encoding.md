# AIE MC Encoding and Decoding

Scope analyzed:
- Encoding: `AIEBaseMCCodeEmitter.*`, `AIEMCCodeEmitter.cpp`, `AIE2MCCodeEmitter.cpp`, `aie2p/AIE2PMCCodeEmitter.cpp`, `AIEBaseAsmBackend.*`, `AIE{1,2,2P}AsmBackend.cpp`
- Format metadata: `AIEFormat.*`, `AIEMCFormats.*`, `AIEBaseMCFormats.cpp`
- Decoding: `AIEBaseDisassembler.h`, `AIE{,2,2P}Disassembler.cpp`

## 1. Encoding model

## 1.1 Endianness and packet writes

`AIEBaseMCCodeEmitter::encodeInstruction`:
- obtains full instruction/packet bit encoding via generated `getBinaryCodeForInstr`,
- emits in 16-bit chunks,
- writes each chunk little-endian to output buffer.

So byte order in object code is little-endian, even though many format/fixup offsets are tracked in big-endian bit indexing for metadata consistency.

## 1.2 Format-driven operand to fixup mapping

For expression operands (`MO.isExpr()`), encoder does:
1. find operand index,
2. read relocatable field list from `MCFormatDesc` (`getFieldsCoveredByOpIdx`),
3. convert fields to `FixupField{Offset,Size}`,
4. pick fixup by `(fields, format-size, signedness)`,
5. emit `MCFixup` and leave relocation-controlled bits zero-filled until link-time patching.

This is the central AIE binary-interface rule: relocation targeting is tied to precise format-field coverage, not only to opcode class.

## 1.3 Composite packet fixup translation

When encoding nested sub-instructions inside a packet:
- encoder first encodes standalone sub-instruction and collects its fixups,
- then translates each fixup field offset from sub-instruction slot coordinates into composite-packet coordinates,
- reselects translated fixup kind for the packet format size.

Translation formula in code (conceptually):
- `TranslatedOffset = RelocField.Offset - SlotOffsetInSubInstruction + SlotOffsetInComposite`

This is required for relocations in VLIW packets where sub-ops occupy non-zero slot offsets.

## 1.4 Slot extraction and packet legality metadata

`AIEInstFormat` and `AIEPacketFormat` provide:
- slot kinds,
- per-slot offsets,
- packet coverage legality (`VLIWFormat::covers`, `PacketFormats::getFormat*`),
- slot NOP opcode (`MCSlotInfo::getNOPOpcode`).

`AIEBaseMCFormats::isFormatAvailable(SlotSet)` gates whether a slot set has a legal format.

## 2. Immediate/fixed-step encoding decisions

Generic fixed-step immediate encoder helper (`getSImmOpValueXStep`) supports:
- signed/unsigned,
- implicit-negative forms,
- divisibility by step,
- range checks before right-shifting out fixed zero bits.

Disassembler mirror (`decodeSImmOperandXStep`) reconstructs signed/unsigned values with fixed zero-bit expansion, including implicit-negative form handling.

This establishes exact encoding semantics for stride-scaled immediates.

## 3. Decoder table structure and width dispatch

## 3.1 AIE1 width dispatch (`AIEDisassembler::getInstruction`)

Instruction size is inferred from low prefix bits of `Bytes[0]`:
- `..01` -> 2 bytes
- `.011` -> 4 bytes
- `0111` -> 8 bytes
- `1111` -> 12 bytes
- `...0` -> 16 bytes

Decoder tables used by width:
- `DecoderTable16/32/64/96/128`
- fallback tables (`DecoderTableFallback*`) on failure.

## 3.2 AIE2/AIE2P width dispatch (`formatSize`)

Size determined by `(Bytes[0] & 0xF)`.

AIE2 mapping:
- nibble `0,2,4,6,8,a,c,e` -> 16
- `1` -> 2
- `3` -> 8
- `5` -> 6
- `7` -> 12
- `9` -> 4
- `b` -> 10
- `d` -> 6
- `f` -> 14

AIE2P mapping:
- odd nibbles `1,3,5,7,9,b,d,f` -> 16
- `0` -> 2
- `2` -> 8
- `4,c` -> 6
- `6` -> 12
- `8` -> 4
- `a` -> 10
- `e` -> 14

Width-specific decode tables:
- `DecoderTableFormats16/32/48/64/80/96/112/128`
- APInt path used for >64-bit formats.

## 4. Slot-level decode design

AIE2/AIE2P disassemblers define slot decoders (Lda/Ldb/Alu/St/Mv/Vec/Lng/Nop) that:
- decode per-slot payload with slot-specific decoder tables,
- wrap each decoded slot as nested `MCInst` operand in a composite instruction.

This mirrors composite packet encoding on the emitter side.

## 5. NOP binary encodings used for padding/alignment

Asm backends implement concrete NOP bytes for code fill:
- AIE1: 2-byte NOP (`01 00`) and optional 4-byte pattern path.
- AIE2: explicit encodings for 2/4/6/8/10/12/14/16-byte nop packets.
- AIE2P: explicit encodings for 2/4/6/8/10/12/14/16-byte nop packets.

`AIETargetELFStreamer::finish()` + `emitCodeAlignment(Align(16))` depends on these encodings to ensure legal executable padding.

## 6. Binary interface summary

The MC layer exposes the following stable binary contract points:
- little-endian serialized instruction bytes,
- nibble/prefix-driven variable packet-size decode rules,
- slot-composite nesting semantics for packet instructions,
- fixup/reloc selection tied to exact field slices and format width,
- arch-specific NOP fill encodings used for section alignment.

Any tool reproducing AIE object generation must match these rules exactly.
