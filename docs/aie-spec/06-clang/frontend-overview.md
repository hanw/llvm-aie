# Clang Frontend AIE Support Overview

## Scope
Primary files:
- `clang/lib/Basic/Targets/AIE.h` / `.cpp`
- `clang/lib/Basic/Targets.cpp`
- `clang/lib/Driver/ToolChains/AIE.h` / `.cpp`
- `clang/lib/Driver/ToolChains/Arch/AIE.h` / `.cpp`
- `clang/lib/Driver/Driver.cpp`
- `clang/lib/CodeGen/Targets/AIE.cpp`
- `clang/lib/CodeGen/CGBuiltin.cpp`
- `clang/lib/CodeGen/CGExprScalar.cpp`
- `clang/include/clang/Basic/TargetBuiltins.h`
- `clang/include/clang/Basic/BuiltinsAIE.def`
- `clang/include/clang/Basic/BuiltinsAIE2P.def`
- `clang/include/clang/Basic/AIETypes.def`

## Class Structure

### Target Info
A single class `AIETargetInfo : public TargetInfo` handles all three AIE variants. Variant detection uses static helpers on the triple:
- `isAIE1(Triple)` — `Triple::aie`
- `isAIE2(Triple)` — `Triple::aie2`
- `isAIE2P(Triple)` — `Triple::aie2p`

### Driver/Toolchain
- `AIEToolChain : public Generic_ELF` — main toolchain class
- `aie::Linker : public Tool` — linker invocation wrapper

### CodeGen ABI
- `AIEABIInfo : public DefaultABIInfo` — argument/return classification
- `AIETargetCodeGenInfo : public TargetCodeGenInfo` — CodeGen hooks

## Target Triples

| Triple | Architecture |
|--------|-------------|
| `aie-none-unknown-elf` | AIE1 (legacy) |
| `aie2-none-unknown-elf` | AIE2 |
| `aie2p-none-unknown-elf` | AIE2P |

Wired in:
- `clang/lib/Basic/Targets.cpp`: `AIETargetInfo` selected for `Triple::aie`, `aie2`, `aie2p`
- `clang/lib/Driver/Driver.cpp`: `AIEToolChain` selected for the same arch set

## Predefined Macros

### Target-defined (getTargetDefines)
Always defined for all variants:
- `__aie__`
- `__AIENGINE__`
- `__AIECC__`
- `__PEANO__`
- `__ELF__`

### Driver-added
- `-D__AIENGINE__` (redundant with target)
- `-D__AIEARCH__=10` (AIE1), `=20` (AIE2), `=21` (AIE2P)

### Via auto-included intrinsic headers
- AIE2: `aiev2intrin.h` → `aiev2_defines.h` defines `__AIE2__`
- AIE2P: `aie2pintrin.h` → `aie2p_defines.h` defines `__AIE2P__`

## Data Layout

All three variants share the same data layout string:
```
e-m:e-p:20:32-i1:8:32-i8:8:32-i16:16:32-i32:32:32-f32:32:32-i64:32-f64:32-a:0:32-n32
```

Key properties:
- Little-endian, ELF mangling
- **20-bit pointers** stored in 32-bit slots
- All sub-32-bit types padded to 32-bit alignment
- Native integer width: 32

## Type Sizes and Alignment

### Basic Types

| Property | Value |
|----------|-------|
| `LongLongAlign` | 32 bits |
| `SuitableAlign` | 32 bits |
| `DoubleAlign` | 32 bits |
| `LongDoubleAlign` | 32 bits |
| `SizeType` | `UnsignedInt` |
| `PtrDiffType` | `SignedInt` |
| `IntPtrType` | `SignedInt` |
| `UseZeroLengthBitfieldAlignment` | `true` |

### Vector Alignment

| Variant | MaxVectorAlign |
|---------|---------------|
| AIE1, AIE2 | 256 bits (32 bytes) |
| AIE2P | 512 bits (64 bytes) |

### Per-Variant Feature Support

| Feature | AIE1 | AIE2 | AIE2P |
|---------|------|------|-------|
| `hasBFloat16Type()` | No | Yes | Yes |
| `hasInt128Type()` | No | Yes | No |
| `hasBitIntType()` | Yes | Yes | Yes |
| `isCLZForZeroUndef()` | Yes | No | No |

### BFloat16 (AIE2/AIE2P only)
- Width: 16 bits, Align: 16 bits
- Format: `llvm::APFloat::BFloat()`
- Mangling: `"8bfloat16"` (vendor extension type)

### Custom Accumulator Types (AIETypes.def)

| Type | ID | Size (bits) | Align (bits) | Variants |
|------|----|-----------:|------------:|----------|
| `__acc32` | ACC32 | 32 | 32 | AIE2/AIE2P |
| `__acc48` | ACC48 | 48 | 64 | AIE1 |
| `__acc64` | ACC64 | 64 | 64 | AIE2/AIE2P |
| `__accfloat` | ACCFLOAT | 32 | 32 | AIE2/AIE2P |

Builtin signature encoding for these types:
- `n` → acc32
- `Ln` → acc48
- `LLn` → acc64
- `g` → accfloat

## Address Space Model

Default address space (`LangAS::Default`) is a superset of all target-specific address spaces:
```cpp
bool isAddressSpaceSupersetOf(LangAS A, LangAS B) const override {
  return A == LangAS::Default && isTargetAddressSpace(B);
}
```

## ABI and CodeGen Integration

### Return Value Classification

1. **Void**: Ignored
2. **Aggregates**:
   - ≤ 128 bits: direct return in registers
   - Types with `AIE2ReturnInRegistersAttr`: always in registers (sparse types)
   - Larger: indirect return via pointer
3. **Enum types**: treated as underlying integer
4. **BitInt > 128 bits**: indirect
5. **Vector types with accumulator elements**: marked `InReg`
6. **Promotable integers**: sign/zero extended

### Argument Classification

1. **Record types**:
   - `RAA_Indirect`: pass by pointer (no ByVal)
   - `RAA_DirectInMemory`: pass by value on stack (ByVal=true)
   - Sparse types (`AIE2IsSparseAttr`): stack alignment forced to 32 bytes
2. **Enum types**: treated as underlying integer
3. **Promotable integers**: extended
4. **All types**: `CanBeFlattened=false` (compound types kept intact)
5. **Vector types with accumulator elements** (AIE2P): marked `InReg` for ACC32, ACCFLOAT, ACC64 element types

### Accumulator Operator Overloads

`CGExprScalar.cpp` implements custom `+` and `-` operators for accumulator vector types:

| Type | Size | `+` Intrinsic | `-` Intrinsic | Config |
|------|------|--------------|--------------|--------|
| `v32acc32` | 1024-bit | `llvm.aie2.add.acc` | `llvm.aie2.sub.acc` | I32 (0x0) |
| `v16acc64` | 1024-bit | `llvm.aie2.add.acc` | `llvm.aie2.sub.acc` | I64 (0x2) |
| `v16accfloat` | 512-bit | `llvm.aie2.add.accfloat` | `llvm.aie2.sub.accfloat` | FP32 (0x1C) |

This allows natural C++ syntax (`acc1 + acc2`) instead of explicit builtin calls.

### Builtin Emission

Dispatch by architecture in `CGBuiltin.cpp`:
- `EmitAIE1BuiltinExpr` / `getAIE1IntrinsicFunction`
- `EmitAIE2BuiltinExpr` / `getAIE2IntrinsicFunction`
- `EmitAIE2PBuiltinExpr` / `getAIE2PIntrinsicFunction`

Most builtins map directly to `llvm::Intrinsic::aie2_*` / `aie2p_*`. Approximately 90 builtins (~14%) require custom lowering — see `builtins.md` for details.

## Driver and Toolchain Defaults

### Default Flags (addClangTargetOptions)

| Flag | Purpose |
|------|---------|
| `-fno-use-init-array` | Disable .init_array sections |
| `-mllvm -vectorize-loops=false` | Disable loop vectorizer (unless `-fvectorize`) |
| `-mllvm -vectorize-slp=false` | Disable SLP vectorizer (unless `-fslp-vectorize`) |
| `-mllvm --two-entry-phi-node-folding-threshold=10` | PHI folding tuning |
| `-fno-threadsafe-statics` | No thread-safe statics (unless explicitly enabled) |
| `-mllvm -mandatory-inlining-before-opt=false` | Inline after opt |
| `-mllvm -basic-aa-full-phi-analysis=true` | Extended alias analysis |
| `-mllvm -basic-aa-max-lookup-search-depth=10` | AA search depth |
| `-mllvm -enable-loop-iter-count-assumptions=true` | Loop assumptions |
| `-fno-builtin-memset/memcpy/memmove` | Disable mem builtins (unless `-fbuiltin`) |
| `-Wno-missing-template-arg-list-after-template-kw` | Suppress warning |

### Intrinsic Header Auto-Include

| Variant | Header | Disabled by |
|---------|--------|-------------|
| AIE1 | `aiev1intrin.h` | `-mno-vitis-headers` |
| AIE2 | `aiev2intrin.h` | `-mno-vitis-headers` |
| AIE2P | `aie2pintrin.h` | `-mno-vitis-headers` |

### Linker Integration

- Linker tool: `ld.lld`
- Always adds `--nmagic` (disables default page alignment)
- Runtime libraries (unless `-nostdlib`/`-nodefaultlibs`): compiler-rt builtins, `-lc`, `-lm`
- Startup files (unless `-nostartfiles`/`-r`): `crt0.o`, `crt1.o`
- C++ standard library: `libc++`
- Runtime library: `compiler-rt`
- Max DWARF version: 4

### Include Path Setup

1. Auto-include intrinsic header (architecture-specific)
2. `-nostdsysteminc` (blocks /usr/include)
3. Target-specific includes: `<driver-dir>/../include/<triple>/`
4. libc++ headers (AIE2/AIE2P only, not AIE1):
   - Per-target: `<driver-dir>/../include/<triple>/c++/<version>/`
   - Generic: `<driver-dir>/../include/c++/<version>/`

## Architecture Features

- `getAIETargetFeatures()`: validates `-march=` value (must be lowercase), handles `OPT_m_aie_Features_Group`
- `getAIEABI()`: returns ABI string from `-mabi=`, default `"ilp32"`
- Register names: `r0-r15`, `sp`, `lr` (18 total)
- Va list kind: `VoidPtrBuiltinVaList`
- No custom inline assembly constraint support
- No GCC register aliases

## Key Design Decisions

- **Single TargetInfo class**: One `AIETargetInfo` handles AIE1/AIE2/AIE2P via triple checks, avoiding class explosion.
- **Auto-included intrinsic headers**: Large builtin surface exposed through auto-included headers rather than requiring explicit includes. Disabled via `-mno-vitis-headers`.
- **Conservative driver defaults**: Vectorizers disabled, mem builtins disabled — AIE has custom vector/memory handling via intrinsics.
- **Accumulator types as first-class**: Custom accumulator types (`__acc32`, `__acc64`, `__accfloat`) with operator overloads enable natural C++ syntax for DSP operations.
- **20-bit pointer model**: Pointers are 20 bits stored in 32-bit slots across all variants, reflected in data layout and address manipulation builtins that truncate/extend between i32 and i20.
