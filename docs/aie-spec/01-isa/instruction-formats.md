# Instruction Formats and Encoding Model

Sources:
- `llvm/lib/Target/AIE/AIEBaseInstrFormats.td`
- `llvm/lib/Target/AIE/AIE2Slots.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PSlots.td`
- `llvm/lib/Target/AIE/AIE2PSlotInclude.td`
- `llvm/lib/Target/AIE/AIE2InstrFormats.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PInstrFormats.td`
- `llvm/lib/Target/AIE/AIE2GenInstrFormats.td`
- `llvm/lib/Target/AIE/AIE2MultiSlotPseudoInstrInfo.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PMultiSlotPseudoInstrInfo.td`
- `llvm/lib/Target/AIE/AIE2RegOperandDef.td`
- `llvm/lib/Target/AIE/AIE2PRegOperandDef.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PImmOperands.td`
- `llvm/lib/Target/AIE/AIE2CompositeFormats.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PCompositeFormats.td`
- `llvm/lib/Target/AIE/AIE2PCompositeFormatsInclude.td`
- `llvm/lib/Target/AIE/aie1/AIE1InstrFormats.td`
- `llvm/lib/Target/AIE/aie1/AIE1CompositeFormats.td`

## 1. Base Format Classes

### Class Hierarchy

```
Instruction (LLVM base)
  └─ InstFormat (LLVM base)
       └─ AIEBaseInst (AIEBaseInstrFormats.td)
            ├─ Size = 4 (default)
            ├─ OutOperandList, InOperandList, AsmString
            ├─ Pattern (for ISel matching)
            ├─ isComposite = false (default)
            ├─ hasCompleteDecoder = false
            ├─ dontcare: bits<32> = 0 (scratchpad for unconstrained bits)
            │
            ├─ AIE2Inst (per-slot instruction base)
            │    └─ AIE2CompositeInst (composite bundle base)
            │         ├─ isComposite = true
            │         ├─ hasSideEffects = 0, mayLoad = 0, mayStore = 0
            │         ├─ AsmVariantName = "NonParsable"
            │         ├─ DecoderNamespace = "Formats"
            │         └─ AIE2_instr*_Composite (8 width classes)
            │              └─ AIE2__instr*__<slot_combo> (58 specializations)
            │
            └─ AIEPseudoInstrAttributes
                 ├─ isPseudo = true
                 ├─ isCodeGenOnly = true
                 └─ SplitPseudo (operand-expansion for tied-subregister/addressing)
```

### Key Properties

- `isComposite`: distinguishes bundle formats from per-slot instruction defs
- `hasCompleteDecoder = false`: globally set because overlapping encodings across slots require slot-scoped decoding
- `dontcare`: 32-bit scratch field for bits that are unconstrained in a given encoding

### Pseudo Instruction Support

- `AIEPseudoInstrAttributes`: marks codegen-only pseudo ops (`isPseudo=1`, `isCodeGenOnly=1`)
- `SplitPseudo`: for operand expansion, notably tied-subregister/addressing decomposition (e.g., expanding 2D/3D address generator state)
- `MultiSlot_Pseudo` (see section 7): pseudo ops that can materialize to different concrete slots

## 2. Slot-Local Encodings

Each physical slot has its own instruction base class and bitfield width. Slot assignment is embedded in the opcode definition itself — not determined post-hoc by packetization.

### AIE2 Slot-Local Classes (`AIE2Slots.td`)

| Slot Class | Field | Width | Size | DecoderNamespace |
|-----------|-------|-------|------|------------------|
| `AIE2_inst_lda_instr32` | `lda` | 21 bits | 4 bytes | `"lda"` |
| `AIE2_inst_ldb_instr32` | `ldb` | 16 bits | 4 bytes | `"ldb"` |
| `AIE2_inst_alu_instr32` | `alu` | 20 bits | 4 bytes | `"alu"` |
| `AIE2_inst_mv_instr32` | `mv` | 22 bits | 4 bytes | `"mv"` |
| `AIE2_inst_st_instr32` | `st` | 21 bits | 4 bytes | `"st"` |
| `AIE2_inst_vec_instr32` | `vec` | 26 bits | 4 bytes | `"vec"` |
| `AIE2_inst_lng_instr48` | `lng` | 42 bits | 6 bytes | `"lng"` |
| `AIE2_inst_nop_instr16` | `nop` | 1 bit | 2 bytes | `"nop"` |

### AIE2P Slot-Local Classes (`aie2p/AIE2PSlots.td`)

| Slot Class | Field | Width | Size | DecoderNamespace |
|-----------|-------|-------|------|------------------|
| `AIE2P_inst_lda_instr32` | `lda` | 20 bits | 4 bytes | `"lda"` |
| `AIE2P_inst_ldb_instr32` | `ldb` | 17 bits | 4 bytes | `"ldb"` |
| `AIE2P_inst_alu_instr32` | `alu` | 20 bits | 4 bytes | `"alu"` |
| `AIE2P_inst_mv_instr32` | `mv` | 22 bits | 4 bytes | `"mv"` |
| `AIE2P_inst_st_instr32` | `st` | 20 bits | 4 bytes | `"st"` |
| `AIE2P_inst_vec_instr32` | `vec` | 26 bits | 4 bytes | `"vec"` |
| `AIE2P_inst_lng_instr48` | `lng` | 42 bits | 6 bytes | `"lng"` |
| (nop via `AIE2PSlotInclude.td`) | `nop` | 1 bit | 2 bytes | `"nop"` |

### Mechanics

- Slot classes bind `let Slot = ...` and `DecoderNamespace = slot.SlotName`
- Nominal instruction size defaults to 4 bytes; long slot uses 6, nop slot uses 2
- Slot-scoped decoder namespaces prevent encoding conflicts among similarly shaped bitfields across different functional units

## 3. Instruction Bit Layout Classes

Generated format classes in `AIE2GenInstrFormats.td` encode per-operation bit layouts with explicit fields.

### Representative AIE2 Bit Layouts

| Format Class | Fields | Slot |
|-------------|--------|------|
| `AIE2_add_r_ri_inst_alu` | `{mRx0, mRx, c7s, 0b11, 0b0}` | alu |
| `AIE2_dms_sts_inst_st` | `{ag_all, mSclSt, 0b1}` | st |
| `AIE2_dmw_ldb_inst_ldb` | `{aga_memw, dst, 0b01}` | ldb |

Vector/MAC formats encode mode bits (`amode/bmode/cmode`, sign, shift, etc.) in dedicated fields within the slot payload.

AIE2P follows the same scheme via generated instruction definitions (`AIE2PGenInstrInfo.td`) and slot classes.

## 4. Composite/Bundle Formats

Composite classes (`isComposite = true`) form variable-width VLIW bundles from slot payloads.

### Encoding Pattern

All composites follow the pattern:

```
Inst = {slot_payloads_concatenated, tag_bits}
```

Tag bits occupy the least significant positions and uniquely identify bundle width.

### Tag Encoding Table

| Width | Inst Width | Payload Width | Tag Value | Tag Width |
|-------|-----------|--------------|-----------|-----------|
| 128-bit | 128 | 127 | `0b0` | 1 |
| 112-bit | 112 | 108 | `0b1111` | 4 |
| 96-bit | 96 | 92 | `0b0111` | 4 |
| 80-bit | 80 | 76 | `0b1011` | 4 |
| 64-bit | 64 | 60 | `0b0011` | 4 |
| 48-bit | 48 | 45 | `0b101` | 3 |
| 32-bit | 32 | 28 | `0b1001` | 4 |
| 16-bit | 16 | 12 | `0b0001` | 4 |

Example (from `AIE2CompositeFormats.td`):
```tablegen
class AIE2_instr128_Composite<dag slot_ins> : AIE2CompositeInst<slot_ins> {
  bits<128> Inst;
  bits<127> instr128;
  let Inst = {instr128, 0b0};
  let Size = 16;
}
```

### Slot Concatenation Order

Slots are concatenated from most significant bits (left) to least significant bits (right). The `alu_mv` region can contain either `{lng, 0b0}` (long slot + 1-bit tag) or `{alu, mv}` (ALU + move), providing a substitution point within composites.

### AIE2 Composite Format Census (58 total)

| Width | Count | Representative Combinations |
|-------|-------|----------------------------|
| 128-bit | 2 | `ldb+lda+st+lng+vec`, `ldb+lda+st+alu+mv+vec` |
| 112-bit | 8 | `st+ldb+lng+vec`, `lda+ldb+alu+mv+vec`, ... |
| 96-bit | 20 | `lda+ldb+alu+st`, `st+alu+mv+vec`, ... |
| 80-bit | 18 | `lda+st+mv`, `ldb+lng`, `st+alu+vec`, ... |
| 64-bit | 12 | `lda+ldb`, `st+vec`, `alu+mv`, `lng`, ... |
| 48-bit | 9 | `lda+alu`, `ldb+st`, `ldb+mv`, `lng`, ... |
| 32-bit | 6 | `vec`, `st`, `mv`, `ldb`, `lda`, `alu` (single-slot) |
| 16-bit | 1 | `nop` only |

### Design Properties

- Low tag bits identify format family/width for the hardware decoder
- `nop_slot` is used intentionally to select larger equivalent formats (alignment/disambiguation)
- Composite classes define intermediate fields and explicit concatenation order
- The 128-bit format is the maximum bundle width, packing all 6 functional slots

## 5. Slot Assignment Rules (Practical View)

By construction, instruction-form classes constrain opcode families to slots:

| Instruction Suffix | Slot | Description |
|-------------------|------|-------------|
| `*_inst_alu` | ALU | Scalar arithmetic/logic |
| `*_inst_lda` | LDA | Load path A |
| `*_inst_ldb` | LDB | Load path B |
| `*_inst_st` | ST | Store operations |
| `*_inst_mv` | MV | Move/misc operations |
| `*_inst_vec` | VEC | Vector operations |
| `*_inst_lng` | LNG | Long (42-bit) instructions |

This is stronger than post-hoc packetization: slot legality is embedded in the opcode definition itself.

## 6. Operand and Immediate Encoding Definitions

Two additional layers constrain format legality:

### Register Operand Wrappers

`OP_*` wrappers in `AIE2RegOperandDef.td` and `AIE2PRegOperandDef.td` bind operand positions to specific register subsets and custom encoder methods. This is how many architectural constraints (e.g., "only r26 can be used as the lock register operand") are made explicit.

### Immediate Classes

Immediate operands (`AIEBaseInstrInfo.td`, `AIE2PImmOperands.td`) encode sign/width/scale constraints:

| Category | Examples | Description |
|----------|---------|-------------|
| Unsigned | `txxu<2>` .. `txxu<20>` | N-bit unsigned |
| Signed | `txxs<3>` .. `txxs<32>` | N-bit signed |
| Negated | `txxsn<N>` | Sign-negated (for subtraction encoding) |
| Step-scaled | `immx4`, `immx8`, `immx16`, `immx32`, `immx64`, `immx128` | Scaled by 4/8/16/32/64/128 bytes |

Step-scaled immediates are critical for memory operations where address offsets must be naturally aligned (e.g., `immx32` for 32-byte-aligned vector accesses, `immx128` for 128-byte-aligned 1024-bit vector accesses).

## 7. Multi-Slot Pseudo Encoding Contract

### Definition Structure

```tablegen
class MultiSlot_Pseudo<dag outs, dag ins, string opcodestr = "",
             string argstr = "", list<AIE2Inst> insts = []>
    : AIE2Inst<outs, ins, opcodestr, argstr> {
  let isPseudo = 1;
  bits<1> isMultiSlotPseudo = 1;
  list<AIE2Inst> materializableInto = insts;
  let isCodeGenOnly = 1;
  let isComposite = 0;
}
```

Key fields:
- `isMultiSlotPseudo`: distinguishes from regular pseudos
- `materializableInto`: list of concrete slot alternatives
- `isCodeGenOnly`: only exists during code generation, not in final encoding

### Representative Multi-Slot Pseudo Definitions

**Pointer arithmetic (PADD):**
- `PADD_mod_pseudo` → `[PADDB_ldb_ptr_inc_nospill_nrm, PADDS_st_ptr_inc_idx, PADDA_lda_ptr_inc_idx]`
- `PADD_imm9_pseudo` → `[PADDB_ldb_ptr_inc_nrm, PADDS_st_ptr_inc_idx_imm, PADDA_lda_ptr_inc_idx_imm]`
- `PADD_imm10_pseudo` → `[PADDA_lda_ptr_inc_idx_imm, PADDS_st_ptr_inc_idx_imm]`
- `PADD_sp_imm_pseudo` → `[PADDB_sp_imm, PADDA_sp_imm]`

**Scalar moves (MOV):**
- `MOV_SCL_pseudo` → `[MOV_mv_scl, MOV_OR]` (via OR encoding trick)
- `MOV_RLC_imm10_pseudo` → `[MOVA_lda_cg, MOVX_alu_cg, MOV_mv_cg, MOVXM_lng_cg]`
- `MOV_PD_imm10_pseudo` → `[MOVA_lda_cg, MOV_mv_cg, MOVXM_lng_cg]`
- `MOV_S_imm10_pseudo` → `[MOV_mv_cg, MOVXM_lng_cg]`

**Vector loads (VLD):**
- `VLD_idx_pseudo` → `[VLDB_dmw_ldb_ag_idx, VLDA_dmw_lda_w_ag_idx]`
- `VLD_pstm_pseudo` → `[VLDB_dmw_ldb_ag_pstm_nrm, VLDA_dmw_lda_w_ag_pstm_nrm]`
- `VLD_2D_pseudo` → `[VLDB_2D, VLDA_2D_dmw_lda_w]`
- `VLD_3D_pseudo` → `[VLDB_3D, VLDA_3D_dmw_lda_w]`

### Design Invariant

All concrete opcode alternatives for a multi-slot pseudo must have **equivalent scheduling itineraries and operand-latency behavior**. This means slot selection can vary for packetization pressure without changing semantic timing assumptions used by the scheduler/resource legality checks.

## 8. Why This Encoding Model Is Architecturally Distinct

1. **Encoding is hierarchical, not flat.**
   First define slot payload encodings, then legal composites. The decoder dispatches on tag bits to determine bundle width, then extracts per-slot payloads at fixed offsets.

2. **Variable bundle width is first-class.**
   A standard fixed-width RISC decoder model does not capture this structure. Bundle width ranges from 16 to 128 bits.

3. **Decoder namespaces are slot-scoped.**
   Distinct slot decoders avoid conflicts among similarly shaped bitfields. Each slot has its own decoder namespace.

4. **Multi-slot pseudos bridge legality and codegen flexibility.**
   Selection can stay abstract while final materialization picks concrete slot placements. The invariant of equivalent itineraries across alternatives guarantees correctness.

5. **Operand encoding is two-layered.**
   Format classes encode bit layout; operand/immediate `.td` files encode admissible value domains for those bitfields. This separation enables the same bit layout to be reused across instructions with different operand constraints.
