# Register File Specification (From TableGen)

Coverage note:
- Register analysis includes base + `aie1` + `aie2` + `aie2p` register/bank/calling-convention `.td` files.
- Content below focuses on the `aie2`/`aie2p` register model that drives the current backend architecture.
- Full enumerations of generated physical-register members and aliases are sourced from `AIE2GenRegisterInfo.td` / `aie2p/AIE2PRegisterInfo.td`; this document focuses on architectural class structure and ABI-relevant groupings.

Sources:
- `llvm/lib/Target/AIE/AIEBaseRegisterInfo.td`
- `llvm/lib/Target/AIE/AIE2RegisterInfo.td`
- `llvm/lib/Target/AIE/AIE2GenRegisterInfo.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PRegisterInfo.td`
- `llvm/lib/Target/AIE/AIEBaseRegisterBanks.td`
- `llvm/lib/Target/AIE/AIE2RegisterBanks.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PRegisterBanks.td`
- `llvm/lib/Target/AIE/AIE2RegOperandDef.td`
- `llvm/lib/Target/AIE/AIE2PRegOperandDef.td`
- `llvm/lib/Target/AIE/AIE2CallingConv.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PCallingConv.td`

## 1. Base Infrastructure

`AIEBaseRegisterClass` (`AIEBaseRegisterInfo.td`) sets common register properties:
- `DecodeZeroBitOperand = true` — enables decoding of zero-bit operands
- `hasCompleteDecoder = false` — disables complete decoder to avoid conflicts between instructions with overlapping encodings

`AIEBaseRegisterOperand` provides automatic encoder method generation based on class name, also with `hasCompleteDecoder = false`.

## 2. Register Class Topology

### AIE2 Scalar/Address Classes

| Class | Width | Count | Members | Purpose |
|-------|-------|-------|---------|---------|
| `eR` | 32-bit | 32 | r0–r31 | General purpose registers |
| `eL` | 64-bit | 8 | l0–l7 (pairs: r16:r17 .. r30:r31) | Paired scalar-64 registers |
| `eP` | 20-bit | 8 | p0–p7 | Pointer registers |
| `eS` | 32-bit | 4 | s0–s3 | Shift registers |
| `eM` | 20-bit | 8 | m0–m7 | Modifier registers |
| `eDC` | 20-bit | 8 | dc0–dc7 | DMA dimension count |
| `eDJ` | 20-bit | 8 | dj0–dj7 | DMA dimension jump/stride |
| `eDN` | 20-bit | 8 | dn0–dn7 | DMA dimension size |
| `eD` | 80-bit | 8 | d0–d7 | 2D DMA config (mod+size+stride+count) |
| `eDS` | 160-bit | 4 | d0_3d–d3_3d | 3D DMA config (pairs of 2D) |
| `mDm` | 20-bit | composite | eDC+eDJ+eDN+eM | Unified DMA modifier class |
| `eSpecial20` | 20-bit | 6 | LS, DP, lr, LE, SP, CORE_ID | Special registers |

Singleton scalar classes (`eR26`, `eR27`, `eR28`, `eR29`, `eR31`) exist for instructions that encode fixed register operands.

DMA field register classes also have high/low splits (`eDCH`/`eDCL`, `eDJH`/`eDJL`, `eDNH`/`eDNL`, each 4 registers) for instructions that only access half the file.

### AIE2 Vector Classes

| Class | Width | Count | Members | Purpose |
|-------|-------|-------|---------|---------|
| `eWL` | 256-bit | 12 | wl0–wl11 | Vector low (even/odd: `eWLE`/`eWLO`) |
| `eWH` | 256-bit | 12 | wh0–wh11 | Vector high (even/odd: `eWHE`/`eWHO`) |
| `mWm/mWa/mWb/mWs` | 256-bit | 24 | All W registers | Operand-constrained variants |
| `eXE` | 512-bit | 6 | x0,x2,x4,x6,x8,x10 | X even (composed of WL+WH pairs) |
| `eXO` | 512-bit | 6 | x1,x3,x5,x7,x9,x11 | X odd |
| `mXm/mXn/mXv/mXw/mXa/mXs` | 512-bit | 12 | All X registers | Operand-constrained variants |
| `eYs` | 1024-bit | 4 | y2–y5 | Y registers (composed of X pairs) |

### AIE2 Accumulator Classes

| Class | Width | Count | Members | Purpose |
|-------|-------|-------|---------|---------|
| `eAMHH/eAMHL/eAMLH/eAMLL` | 256-bit | 9 each | amhh0–8, amhl0–8, amlh0–8, amll0–8 | AM quarter accumulators |
| `mAMm/mAMs` | 256-bit | 36 | All AM quarters unified | Operand variants |
| `eBMH/eBML` | 512-bit | 9 each | bmh0–8, bml0–8 (composed: amhl+amhh, amll+amlh) | BM half accumulators |
| `eBMSH/eBMSL` | 512-bit | 8 each | bmh0–7, bml0–7 | BM short (8-register subset) |
| `mBMm/mBMs/mBMa` | 512-bit | 18 | eBMH+eBML | Operand-constrained variants |
| `eCM` | 1024-bit | 9 | cm0–cm8 (composed: bml+bmh) | CM full accumulators |
| `mCMm/mCMs/mCMa` | 1024-bit | 9 | eCM | Operand-constrained variants |

### AIE2 Mask and Sparse Classes

| Class | Width | Count | Members | Purpose |
|-------|-------|-------|---------|---------|
| `eQQEs/eQQOs` | 128-bit | 2 each | q0,q2 / q1,q3 | Mask even/odd |
| `mQQm/mQQa/mQQs` | 128-bit | 4 | q0–q3 | Mask operand variants |
| `qwl0–qwl3` | 320-bit | 4 | 64-bit mask + 256-bit vector low | Vector-mask composite |
| `qwh0–qwh3` | 320-bit | 4 | 64-bit mask + 256-bit vector high | Vector-mask composite |
| `eQQXEs/eQQXOs` | 640-bit | 2 each | qx0,qx2 / qx1,qx3 | Sparse vector-mask |
| `SPARSEVEC640` | 640-bit | 4 | All QX | Unified sparse (non-allocatable) |

### AIE2 Control and Status Registers

| Class | Width | Count | Members |
|-------|-------|-------|---------|
| `mCRm` | 32-bit | 12 | crSat, crRnd, crFPMask, crF2IMask, crF2FMask, crSRSSign, crUPSSign, crPackSign, crUnpackSign, crVaddSign, crSCDEn, crMCDEn |
| `mSRm` | 32-bit | 10 | srCarry, srSS0, srMS0, srSRS_of, srUPS_of, srCompr_uf, srSparse_of, srFPFlags, srF2IFlags, srF2FFlags |

### AIE2P Additional Classes

AIE2P shares the scalar/address/vector base structure but extends with:

| Class | Width | Count | Purpose |
|-------|-------|-------|---------|
| `ACC512` (BM) | 512-bit | varies | 512-bit accumulators (bmll, bmlh, bmhl, bmhh variants) |
| `ACC1024` (CM) | 1024-bit | 10 | cml/cmh variants |
| `ACC2048` (DM) | 2048-bit | 5 | dm0–dm4 (composed of CM pairs) |
| `EXPVEC64` | 64-bit | 12 | e0–e11 (exponent registers, with el/eh sub-regs) |
| `VEC288` | 288-bit | varies | 32-bit exponent + 256-bit vector (ewl/ewh composites) |
| `VEC576` (EX) | 576-bit | 12 | 512-bit X + 64-bit E (block floating-point) |
| `VEC1152` (EY) | 1152-bit | varies | Two 576-bit registers (composed EX pairs) |
| `VEC352` | 352-bit | varies | 64-bit mask + 32-bit exp + 256-bit vector |
| `VEC704` | 704-bit | varies | Two 352-bit registers |
| `VEC1408` | 1408-bit | varies | Two 704-bit registers |
| `FIFO512` | 512-bit | varies | Load/store FIFO half (eLdFifoHReg, eLdFifoLReg, mStFifoh, mStFifol) |
| `FIFO1024` | 1024-bit | varies | Combined FIFO (eLdFifoReg, mStFifo) |

## 3. Sub-register and Tuple Structure

Both AIE2 and AIE2P define explicit subregister indices for:

### Scalar Pair Halves
- `sub_l_even`: 32-bit at offset 0
- `sub_l_odd`: 32-bit at offset 32

### DMA 2D/3D Address Generator Fields
- `sub_mod`: 20-bit at offset 0
- `sub_dim_size`: 20-bit at offset 20
- `sub_dim_stride`: 20-bit at offset 40
- `sub_dim_count`: 20-bit at offset 60
- `sub_lo_dim`: 80-bit at offset 0 (for 3D low half)
- `sub_hi_dim`: 80-bit at offset 80 (for 3D high half)

### Vector Slices
- `sub_256_lo/hi`: 256-bit halves of 512-bit vectors
- `sub_512_lo/hi`: 512-bit halves of 1024-bit vectors
- Composed forms: `sub_512_hi_256_lo`, `sub_512_hi_256_hi`

### Mask/Sparse Partitions
- `sub_lo_mask`/`sub_hi_mask`: 64-bit mask halves
- `sub_q`: 64-bit mask at offset 0 (in vector-mask composites)
- `sub_w`: 256-bit vector at offset 64
- `sub_sparse_q`: 128-bit mask at offset 512 (in sparse composites)
- `sub_sparse_x`: 512-bit vector at offset 0

### AIE2P Extended Sub-registers
- Accumulator partitioning: `sub_512_acc_lo/hi`, `sub_1024_acc_lo/hi`
- BFP composition: `sub_bfp_exp` (32-bit at offset 0), `sub_bfp_vec` (256-bit at offset 32)
- BFP-16 composition: `sub_bfp16_x` (512-bit at offset 0), `sub_bfp16_e` (32-bit at offset 512)
- VecMaskExp composites: `sub_q_vecmaskexp_352`, `sub_e_vecmaskexp_352`, `sub_w_vecmaskexp_352`, plus `sub_lo/hi_vecmaskexp_352` and `sub_even/odd_vecmaskexp_704`
- FIFO partitioning: `sub_ptr`, `sub_fifo`, `sub_avail`, plus FIFO hi/lo splits
- Exponent halves: `sub_lo_exp` (32-bit at offset 0), `sub_hi_exp` (32-bit at offset 32)

Many wide classes are declared `CoveredBySubRegs = 1`, making subregister decomposition part of the architectural contract rather than only a compiler convenience.

Design implication:
- Instruction defs can tie or update parts of larger architectural objects (especially 2D/3D pointer/modifier state), enabling compact addressing ops and post-update semantics.

## 4. Register Aliases and Operand-Class Constraints

The backend heavily uses operand-specific register classes (examples: `mR26_lock`, `mR29_insert`, `mSCD_r`, `mMvBMXDst`). The `m`-prefix naming convention indicates operand-constrained subsets.

Why this matters:
- Many instructions are not generic `reg, reg, reg`; operand position determines legal register subset.
- This is a core mechanism for encoding architectural constraints and for driving per-operand scheduling/resource variants.

Operand-wrapper detail:
- `OP_*` wrappers in `AIE2RegOperandDef.td` and `AIE2PRegOperandDef.td` provide a second-level contract beyond register classes: they bind operand positions to tightly-scoped subsets (and occasionally custom encoder hooks), which is how many architectural constraints are made explicit in TableGen.

## 5. Register Banks (GlobalISel)

### Base Banks (`AIEBaseRegisterBanks.td`)
- `PTRRegBank` → `[eP]`
- `MODRegBank` → `[mDm]`
- `VRegBank` → `[VEC128, VEC256, VEC512, VEC1024]`

### AIE2 Banks
- `GPRRegBank` → `[eR, eL]`
- `AccRegBank` → `[ACC256, ACC512, ACC1024]`

### AIE2P Banks
- `GPRRegBank` → `[eR, eL, eE, EXPVEC64]`
- `AccRegBank` → `[ACC512, ACC1024, ACC2048]`
- `FifoRegBank` → `[FIFO512, FIFO1024]`

Design implication:
- GlobalISel regbankselect is not just scalar-vs-vector; AIE2P introduces additional bank domains (notably FIFO and exponent-in-GPR) that affect legal instruction selection and copy insertion.

## 6. Calling Convention Register Assignments

### AIE2 (`AIE2CallingConv.td`)

#### Argument Passing (CC_AIE2)

| Type Domain | Registers | Notes |
|-------------|-----------|-------|
| Pointers | p0–p5 | 6 pointer registers |
| Scalars (i32, f32) | r0–r7 | i1/i8/i16/bf16 promoted to i32 |
| V32 (v4i8, v2i16, v2bf16) | r0–r7 | In GPR |
| V64 (v8i8, v4i16, v2i32, ...) | l0–l7 | In L registers |
| V128 (v16i8, v8i16, v4i32, ...) | wl0–wl11, wh0–wh11 | W registers |
| V256 (v32i8, v16i16, v8i32, ...) | W then AM registers | Extended allocation |
| V512 (v64i8, v32i16, v16i32, ...) | x0–x11 | X registers |
| V1024 (v128i8, v64i16, v32i32, ...) | y2–y5 | Y registers |
| Masks (i128) | q0–q3 | Q registers |
| Acc256 (v4i64) | AM registers, then W | Two-tier fallback |
| Acc512 (v8i64) | BM registers | |
| Acc1024 (v16i64) | CM registers | |

Sparse arguments use custom consecutive-register allocation (`CC_AIE2_SPARSE`).

#### Stack Allocation (overflow)

| Value Width | Size | Alignment |
|-------------|------|-----------|
| 32-bit | 4 | 4 |
| 64-bit | 8 | 4 |
| 128-bit | 16 | 16 |
| 256-bit | 32 | 32 |
| 512-bit | 64 | 32 |
| 1024-bit | 128 | 32 |

#### Callee-Saved (CSR_AIE2)
`lr, r16–r23, p6, p7` (11 registers)

### AIE2P (`AIE2PCallingConv.td`)

#### Argument Passing (CC_AIE2P)

| Type Domain | Registers | Notes |
|-------------|-----------|-------|
| Pointers | p0–p5 | Same as AIE2 |
| Scalars (i32, f32) | r0–r7 | Same as AIE2 |
| L-regs (i64) | l0–l15 (excl. l14) | More L registers than AIE2 |
| V256 | wl0–wl11, wh0–wh11 | W registers |
| V512 | x0–x11 | X registers |
| V1024 | y2–y5 | Y registers |
| Acc512 | bmll0–bmhh4 | 20 registers (4 quarter variants) |
| Acc1024 | cml0–cmh4 | 10 registers |
| Acc2048 | dm0–dm4 | 5 registers (AIE2P only) |
| Masks (i128) | q0–q3 | Q registers |

BFP-16 types use consecutive-register allocation.

Return convention adds `InReg` flag checks for accumulators, allowing larger types to be returned in specialized register files when explicitly annotated.

#### Callee-Saved (CSR_AIE2P)
`lr, r8–r15, p6, p7` (10 registers)

Note: AIE2P callee-saved GPRs are r8–r15 (vs. r16–r23 in AIE2), a significant ABI difference.

## 7. Key Design Decisions

1. **Register classes are semantic, not cosmetic.**
   They encode operand legality, aliasing/subregister behavior, and schedule specialization points. The `m`-prefix operand constraint classes are the primary mechanism.

2. **Subregister composition is architectural.**
   2D/3D addressing and composite vector/BFP operations depend on explicit subregister maps. `CoveredBySubRegs = 1` makes this a correctness requirement.

3. **Calling convention is vector-width aware.**
   ABI placement explicitly tracks vector and accumulator widths; this is far richer than standard scalar-first RISC ABIs. The two-tier fallback (e.g., ACC256 → AM → W) shows cross-domain flexibility.

4. **Operand-specific register classes drive scheduling.**
   The large number of `m*` register class variants directly feeds into itinerary selection, particularly in AIE2P where itinerary classes are keyed by operand register class (`ItineraryRegPairs`).

5. **AIE2P extends data path specialization.**
   FIFO, exponent, and BFP register classes reflect hardware accelerator features for deep learning workloads that have no analog in standard RISC targets.
