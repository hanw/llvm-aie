# AIE Instruction Selection Lowering

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEBaseISelLowering.*`
- `llvm/lib/Target/AIE/AIE2ISelLowering.*`
- `llvm/lib/Target/AIE/aie2p/AIE2PISelLowering.*`
- `llvm/lib/Target/AIE/aie1/AIE1ISelLowering.*` (legacy SelectionDAG path)

## Public Interface (Key Overrides)
`AIEBaseTargetLowering`:
- `getOptimalMemOpType()`
- `getOptimalMemOpLLT()`
- `getVectorIdxTy()`
- `canMergeStoresTo()`
- `getNumRegistersForCallingConv()`
- `getRegisterTypeForCallingConvAssignment()`
- `getRegisterTypeForCallingConv()`
- ABI helpers: `CCAssignFnForCall()`, `CCAssignFnForReturn()`, `alignFirstVASlot()`

`AIE2TargetLowering`:
- `getRegisterTypeForCallingConvAssignment()`
- `getRegisterTypeForCallingConv()`
- `getPreferredVectorAction()`
- `isCheapToSpeculateCtlz()`
- `functionArgumentNeedsConsecutiveRegisters()`

`AIE2PTargetLowering`:
- `getRegisterTypeForCallingConvAssignment()`
- `getRegisterTypeForCallingConv()`
- `getPreferredVectorAction()`
- `isCheapToSpeculateCtlz()`
- `functionArgumentNeedsConsecutiveRegisters()`
- `getTgtMemIntrinsic()`

`AIE1TargetLowering` (legacy DAG lowering):
- `LowerOperation()`
- `LowerFormalArguments()`
- `LowerCall()`
- `CanLowerReturn()`
- `LowerReturn()`
- `ReplaceNodeResults()`
- `getRegForInlineAsmConstraint()`
- `getShiftAmountTy()`

## Custom-Lowered Operations

### AIE2/AIE2P (GlobalISel Path)
Most operation legality and combine behavior is expressed in GlobalISel legalizer + combiner passes, not in monolithic per-op SelectionDAG custom lowering tables.

`AIE2ISelLowering` and `AIE2PISelLowering` mainly provide:
- ABI/type policy (calling convention register assignment, type promotion rules)
- Intrinsic memory-behavior hooks
- Dynamic type registration from TableGen-declared register class/type compatibility

### Type Registration (AIE2)
Legal types are dynamically derived from register classes. The constructor iterates register classes and registers their associated types, **excluding 128-bit vectors** (which receive special ABI treatment instead of being generally legal).

### AIE1 (SelectionDAG Legacy Path)
Explicit custom/expand/libcall legality setup with `setOperationAction` and related hooks.

Notable custom-lowered SDNode ops in `LowerOperation()`:

| Operation | Lowering |
|-----------|----------|
| `BUILD_VECTOR` | Custom vector construction |
| `DYNAMIC_STACKALLOC` | Stack growth handling |
| `GlobalAddress` / `GlobalTLSAddress` / `TargetGlobalAddress` | Address wrapping via `AIEISD::GLOBALADDRESSWRAPPER` |
| `ConstantPool` | Constant pool access |
| vector `LOAD`/`STORE` (`v2i32`) | Scalarization path |
| `VASTART` / `VAARG` | Variadic argument handling |

AIE-specific DAG nodes (`AIEISD::*`) introduced by lowering:
- `CALL`, `TAIL`, `RET_FLAG` — call/return sequences
- `GLOBALADDRESSWRAPPER` — target address formation
- `GPR_CAST` — register class conversion
- `STACK_SAVE`, `STACK_RESTORE` — spill placement

Many scalar ops are expanded/libcalled due to ISA constraints (div/rem, many rotates/shifts, several FP conversions).

## Legal Types and Type Promotion/Expansion Strategy

### Common/Base
- ABI modeled around 32-bit stack slots and 32-bit argument alignment.
- `i64`/`f64` are split across two 32-bit CC registers (`getNumRegistersForCallingConv` returns 2, assignment type `i32`).
- First variadic stack slot is forced to 32-byte alignment via `alignFirstVASlot()`.
- `getOptimalMemOpType()` returns `MVT::i32` — prefers scalar ops to avoid expensive vector register initialization for small copies.
- `getOptimalMemOpLLT()` may choose vector LLTs for larger aligned copies/sets on AIE2/AIE2P.
- Stack slot size: 4 bytes (32 bits).

### AIE2/AIE2P
- Register class legality is derived from TableGen-declared regclass/type compatibility.
- 128-bit vectors are treated specially:
  - Not treated as generally legal vector operation width.
  - Still preserved for ABI differentiation versus 256-bit vectors.
  - Passed in wider register forms via CC-assignment type logic.
- Preferred vector legalization action is widening (`TypeWidenVector`), especially for 128-bit vectors.

## Calling Convention Implementation

### CC Dispatch
- `CCAssignFnForCall(IsVarArg)` selects `CC_AIE*` / `CC_AIE*_Stack`.
- `CCAssignFnForReturn()` selects `RetCC_AIE*`.

### Register Lists

AIE2 argument registers:
- Scalar: `r0`–`r5` (6 registers)
- Pointer: `p0`–`p5` (6 registers)
- Vector: subset of W/X/Y registers
- Accumulator: subset of BM/CM registers

AIE2P argument registers (different allocation):
- Scalar: `r0`–`r7` (8 registers)
- Similar pointer/vector/accumulator allocation with AIE2P-specific classes

### Split-Type Argument Handling (`Handle_Split_Arg`)
1. Allocate contiguous register blocks when available.
2. Otherwise spill as a whole block to stack.
3. Stack assignment accounts for AIE stack model (upward-growing) and ABI ordering.

### Consecutive Register Pairing
Target-specific consecutive register pairing for complex types:

**AIE2 sparse vector+mask:**
- Sparse vector arguments require paired vector + mask registers in consecutive positions.

**AIE2P BFP16 mantissa/exponent:**
- BFP16 576-bit: 12 register pairs (mantissa + exponent)
- BFP16 1056-bit: 6 sets of 4 registers
- Detection via `isBFP16Type()` — checks for specific struct layouts containing mantissa vector + exponent vector fields.

### Tail Call Eligibility
Strict safety checks:
- No varargs.
- No outgoing stack arguments.
- No indirect argument passing.
- No struct-return conflicts.
- Preserved-register mask compatibility between caller/callee.
- Excludes weak external linkage and byval cases.

## Address Space Handling
- In lowering itself, address-space semantics are intentionally lightweight.
- Target-machine layer treats addrspaces as banking annotations and later flattens them.
- This design keeps lowering generic while deferring physical banking placement to dedicated MIR passes.
- On legacy `aie1`, global/TLS address nodes are wrapped explicitly through `AIEISD::GLOBALADDRESSWRAPPER` during lowering to preserve target-specific addressing semantics into instruction selection.

## Intrinsic Lowering

### FIFO Intrinsics (AIE2P)
`AIE2PTargetLowering::getTgtMemIntrinsic()` describes memory effects for FIFO intrinsics:
- Marks FIFO load intrinsics as `MOLoad`, unknown-size (`memVT = Other`), conservative alignment.
- Marks FIFO store/flush intrinsics as `MOStore`, unknown-size, conservative alignment.
- This is a deliberate alias/scheduling safety model: FIFO HW may access ranges larger than explicit IR operands suggest.

## Key Design Decisions
- For modern targets, legalization/selection intelligence is concentrated in GlobalISel legalizer+combiner pipeline, not in monolithic per-op DAG lowering.
- ABI compatibility for special packed/sparse/BFP16 structs is preserved using custom consecutive-register assignment rules.
- Memcpy/memset lowering balances scalar simplicity with optional vectorized LLT choices for aligned bulk operations.
- Type registration is data-driven from register classes rather than hardcoded, ensuring consistency between register file definitions and legal type set.

## Invariants Downstream Code Depends On
- CC split/chunk ordering and stack offsets must preserve ABI bit layout exactly.
- Vararg first stack slot alignment (32 bytes) is assumed by vararg access lowering.
- 128-bit vector ABI treatment must remain distinct from generic vector legalization.
- For AIE2P FIFO intrinsics, conservative unknown-size memory operands are required for safe alias and scheduling behavior.
- For `aie1`, custom AIEISD nodes emitted by lowering (`CALL`/`RET_FLAG`/`GLOBALADDRESSWRAPPER`/`STACK_SAVE`) must remain aligned with DAG-to-DAG pattern expectations.
