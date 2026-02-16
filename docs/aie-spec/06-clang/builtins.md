# AIE Builtins

## Scope
- `clang/include/clang/Basic/BuiltinsAIE.def` — AIE1 + AIE2 builtins
- `clang/include/clang/Basic/BuiltinsAIE2P.def` — AIE2P builtins (included at tail of AIE.def)
- `clang/include/clang/Basic/TargetBuiltins.h` — builtin ID namespace
- `clang/lib/CodeGen/CGBuiltin.cpp` — builtin emission and custom lowering

## Builtin Namespace

All AIE builtins are unified in the `clang::AIE` namespace:
```cpp
namespace AIE {
  enum {
    LastTIBuiltin = clang::Builtin::FirstTSBuiltin - 1,
    #define BUILTIN(ID, TYPE, ATTRS) BI##ID,
    #include "clang/Basic/BuiltinsAIE.def"
    LastTSBuiltin
  };
}
```

## Builtin Counts

| Source File | Prefix | Count | Total |
|-------------|--------|------:|------:|
| BuiltinsAIE.def | `__builtin_aie_*` | ~41 | |
| BuiltinsAIE.def | `__builtin_aiev2_*` | ~395 | ~436 |
| BuiltinsAIE2P.def | `__builtin_aie2p_*` | ~219 | ~219 |
| **All** | | | **~655** |

## Signature Encoding

Signatures use Clang's standard builtin encoding format. AIE adds custom type shorthands:
- `n` → `__acc32`
- `Ln` → `__acc48`
- `LLn` → `__acc64`
- `g` → `__accfloat`

Examples:
- `__builtin_aie_event`: `"vi"` — returns void, takes int
- `__builtin_aie_ctrl_packet_header`: `"iiiii"` — returns int, takes 4 ints
- `__builtin_aiev2_add_2d`: `"v*v*iiii&"` — pointer + int args with by-reference output

## Functional Categories

### AIE1 (`__builtin_aie_*`, ~41 builtins)

| Category | Examples | Count |
|----------|---------|------:|
| Events | `event` | 1 |
| Locks | `acquire`, `release` | 2 |
| Streams | `get_ss`, `put_ms`, `packet_header`, `ctrl_packet_header` | 6 |
| Bit manipulation | `bitget`, `bitset`, `bitget_mc0/mc1`, `bitset_md0/md1` | 6 |
| Scalar math | `sqrt/invsqrt/inv` × `flt_flt/fix_flt/flt_fix/fix_fix` | 12 |
| Vector undef | `v4f32undef`, `v8i48undef`, etc. | 12 |
| FP vector ops | `vfpmul`, `vfpmac`, `vfpsimplemul`, `vfpsimplemac` | 4 |
| Vector update/extract | `upd_v_v8i32_lo/hi`, `ext_w_v16i32_lo/hi` | 9 |
| Concatenation | `concat_v16i16`, `concat_v32i16` | 2 |
| Multiply/MAC | `mul4/8/16`, `mac16` variants | 7 |
| SRS | `bsrs_v16i8`, `ubsrs_v16i8` | 2 |
| Compare/select | `prim_v32int16`, `pack_v16int16` | 4 |

### AIE2 (`__builtin_aiev2_*`, ~395 builtins)

| Category | Examples | Count |
|----------|---------|------:|
| Multiply/MAC | `I512/I1024_mul/mac/msc/negmul` acc32/acc64/accfloat/bf16 variants | ~130 |
| Vector compare/select | `vabs_gtz/vge/vlt/vmax_lt/vmin_ge/vsub_ge/vsub_lt` × 8/16/32/bf16 | ~52 |
| Vector undef | 23 types × various widths | ~52 |
| Vector manipulation | `vshift`, `vinsert`, `vbroadcast`, `vshuffle`, `vextract_elem` | ~60 |
| Update/extract/concat | `upd/set/ext_I64/I256/I512/I1024`, `concat_I512/I1024` | ~36 |
| Sparse operations | `sparse_pop/peek` × 4/8/16/bf × and_get_pointer/set_lo/insert_hi | ~36 |
| Accumulator arithmetic | `add/sub/negadd/negsub_acc/accfloat` | 8 |
| SRS/UPS | `srs` and `ups` for acc32/acc64 × I256/I512 | 12 |
| Streams | `scd_read/expand`, `mcd_write`, `get/put_ss/ms` | ~14 |
| Address manipulation | `add_2d`, `add_3d` | 2 |
| Load operations | `load_4x16/32/64_lo/hi` | 6 |
| Pack/unpack | `pack_I8_I16/I4_I8`, `unpack_I16_I8/I8_I4` | 4 |
| Events/semaphores | `event`, `acquire/release_cond`, `done` | 6 |
| Tile memory | `read_tm`, `write_tm` | 2 |
| Control registers | `set/get_ctrl_reg`, `get_coreid` | 3 |
| Scheduling | `sched_barrier` | 1 |
| Division | `divstep` | 1 |

### AIE2P (`__builtin_aie2p_*`, ~219 builtins)

| Category | Examples | Count |
|----------|---------|------:|
| MAC/MUL operations | ACC2048 variants for add/mac/mul/msc × I512/I1024/bf | ~90 |
| Vector compare/select | Same as AIE2 pattern | ~40 |
| FIFO operations | `fifo_st_push/flush`, `fifo_ld_fill/pop` × 1D/2D/3D × unaligned/bfp16 | ~26 |
| Vector manipulation | `vshift`, `vinsert`, `vbroadcast`, `vshuffle` | ~15 |
| SRS/UPS | acc32/acc64 × v32/v64 | ~10 |
| Type conversions | `accfloat_to_bf16/float`, `bf16_to_accfloat/i32` | 8 |
| Pack/unpack | I512/I1024 × I8_I16/I4_I8 | 8 |
| Streams | scd/mcd + scalar + packet headers | ~10 |
| Non-linear FP | `sqrtf`, `inv`, `invsqrt`, `exp2`, `tanh` | 5 |
| Semaphores | `acquire/release_cond`, `done` | 5 |
| Control/status regs | `set/get_ctrl_reg`, `set/get_status_reg`, `get_coreid` | 5 |
| BFP16 MAC/MUL | BFP576/BFP1152 variants | ~12 |
| BFP16 conversions | `v64accfloat_to_v64bfp16ebs8/16` | 3 |
| Address manipulation | `add_2d`, `add_3d` | 2 |
| Load operations | `load_4x16/32/64_lo/hi` | 6 |
| Tile memory | `read_tm`, `write_tm` | 2 |

## Intrinsic Mapping

### Direct Mapping (1:1)
Most builtins (~86%) map directly to LLVM intrinsics:
- `__builtin_aiev2_vabs_gtz8` → `llvm.aie2.vabs.gtz8`
- `__builtin_aie_ctrl_packet_header` → `llvm.aie.ctrl.packet.header`
- `__builtin_aie2p_sqrtf` → `llvm.aie2p.sqrtf`

Dispatch functions:
- `getAIE1IntrinsicFunction(BuiltinID)` — AIE1 intrinsic lookup
- `getAIE2IntrinsicFunction(BuiltinID)` — AIE2 intrinsic lookup
- `getAIE2PIntrinsicFunction(BuiltinID)` — AIE2P intrinsic lookup

### Custom Lowering (~90 builtins, ~14%)

Builtins requiring custom lowering fall into these patterns:

#### 1. Address Manipulation (i32 ↔ i20 truncation/extension)

Builtins: `add_2d`, `add_3d` (AIE2 and AIE2P)

Pattern:
- Input address increments: truncate i32 → i20
- Output counts: zero-extend i20 → i32
- Returns pointer + count(s) via output parameters

#### 2. Multi-Result Tuple Unpacking

Many intrinsics return LLVM struct types that must be decomposed into separate output parameters.

**Vector operations with condition mask output** (~40 builtins):
- `vabs_gtz8/16/32`, `vbneg_ltz8/16/32`
- `vmaxdiff_lt8/16/32`, `vmax_lt8/16/32/bf16`, `vmin_ge8/16/32/bf16`
- `vneg_gtz8/16/32`, `vsub_ge8/16/32`, `vsub_lt8/16/32`
- Returns: vector result + comparison/condition mask stored to output pointer

**Stream operations with success flag**:
- `put_ms_nb`, `put_ms_nb_packet_header`, `put_ms_nb_ctrl_packet_header`
- Returns: success flag as output parameter

**Division step with dual outputs**:
- `divstep` — returns remainder and quotient to separate output parameters

#### 3. Sparse Operations (triple outputs)

Builtins: `sparse_pop/peek_4/8/16/16_bfloat` × `and_get_pointer/set_lo/insert_hi`

Returns: vector data + mask + pointer

Reset/fill variants return updated pointer.

#### 4. BFP16 Mantissa/Exponent Splitting

Builtins:
- `v64accfloat_to_v64bfp16ebs8/16`
- `v64bfp16ebs8_to_v64bfp16ebs16`
- `vshuffle_576_bfp16`

Returns: mantissa and exponent as separate outputs

#### 5. FIFO Operations (multi-output with address tracking)

**Store** (`fifo_st_push/flush` variants):
- Returns updated pointer, FIFO state, and position
- 2D/3D variants also handle address count zext (i20→i32)

**Load** (`fifo_ld_fill/pop` variants):
- Returns vector data + updated pointer, FIFO state, position
- BFP16 variants split mantissa/exponent
- 2D/3D variants handle count zext

#### 6. Cascade Stream Expand with Increment

Builtins: `scd_expand_ACC1024_incr`, `scd_expand_ACC2048_incr`

Returns: accumulator data + updated position

## Key Design Decisions

- **Unified namespace**: All AIE1/AIE2/AIE2P builtins share one `clang::AIE` namespace. AIE2P builtins are included at the tail of `BuiltinsAIE.def`.
- **Custom lowering is concentrated**: ~14% of builtins need custom lowering. The rest use direct intrinsic mapping via lookup tables.
- **i20 address truncation**: Address manipulation builtins handle the i32→i20 truncation at the frontend level, matching the 20-bit pointer architecture.
- **Multi-result via output parameters**: C builtins can't return multiple values, so multi-result intrinsics use pointer output parameters. The lowering extracts struct elements and stores to these pointers.
- **Category explosion in AIE2/AIE2P**: The builtin count grows primarily from type-width combinatorics (8/16/32/bf16 × operation) and addressing variants (1D/2D/3D).
