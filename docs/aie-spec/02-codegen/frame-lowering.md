# AIE Frame Lowering

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEBaseFrameLowering.*`
- `llvm/lib/Target/AIE/AIE2FrameLowering.*`
- `llvm/lib/Target/AIE/aie1/AIE1FrameLowering.*`
- `llvm/lib/Target/AIE/aie2p/AIE2PFrameLowering.*`

## Public Interface (Key Overrides)
`AIEBaseFrameLowering`:
- `hasFPImpl()`
- `getFrameIndexReference()`
- `hasReservedCallFrame()`
- `eliminateCallFramePseudoInstr()`
- `assignCalleeSavedSpillSlots()`
- `spillCalleeSavedRegisters()`
- `restoreCalleeSavedRegisters()`
- `orderFrameObjectsIncludesCalleeSaves()`
- `emitPrologue()`
- `emitEpilogue()`
- `orderFrameObjects()`

`AIE2FrameLowering`:
- `determineCalleeSaves()`
- target-specific `adjustSPReg()` and `adjustReg()`

`AIE2PFrameLowering`:
- `determineCalleeSaves()`
- `processFunctionBeforeFrameFinalized()`
- target-specific `adjustSPReg()` and `adjustReg()`

## Frame Layout Model

### Stack Growth Direction
Stack direction is explicitly `StackGrowsUp`. This is a key architectural difference from most targets — the stack pointer increases as frames are allocated, not decreases.

### Frame Layout Diagram

```
                        ┌──────────────────────┐
                        │  Caller's Frame       │
  SP (entry) ──────────►├──────────────────────┤
                        │  Callee-saved regs    │
                        ├──────────────────────┤
                        │  Local variables      │
                        │  (sorted by alignment)│
                        ├──────────────────────┤
                        │  Spill slots          │
                        ├──────────────────────┤
                        │  Outgoing call args   │
  SP (after prologue) ──►├──────────────────────┤
                        │  (next frame grows up)│
                        └──────────────────────┘
         ↑ Stack grows upward
```

### Frame Index Reference
Frame index reference formula is based on:
- Object offset within the frame
- Total stack size
- Local area offset
- Offset adjustment

Debug assertions verify:
- Total stack size alignment
- Per-object SP-relative alignment invariants

### Alignment by Target

| Target | Stack Alignment | SP Adjustment Granularity |
|--------|----------------|--------------------------|
| AIE2 | 32 bytes | Multiples of 32 bytes |
| AIE2P | 64 bytes | Multiples of 64 bytes |

## Prologue/Epilogue Generation

### Prologue (`emitPrologue`)
1. Compute and align final frame size (`determineFrameLayout`).
2. Early-exit if no stack allocation is needed.
3. Adjust SP by stack size (`adjustSPReg`).
4. If FP is required:
   - Skip over generated callee-save stores.
   - Copy SP to FP.
   - Adjust FP by `-StackSize`.

### Epilogue (`emitEpilogue`)
1. If variable-sized objects exist:
   - Restore SP from FP path.
   - Then apply stack adjustment.
2. Otherwise deallocate stack directly via SP adjustment.

## Stack Slot Allocation and Ordering
`orderFrameObjects()` sorts fixed-size objects by:
1. Higher alignment first.
2. Then larger size.
3. Then object index (stable tiebreaker).

This minimizes padding and improves immediate-offset accessibility for stack accesses.

## Call Frame and Dynamic Stack Behavior
- `hasFPImpl()` enables FP when:
  - Frame pointer elimination is disabled.
  - Variable-sized objects exist.
  - Frame address is taken.
- `hasReservedCallFrame()` is disabled for var-sized objects or FP usage.
- `eliminateCallFramePseudoInstr()` materializes call-frame setup/destroy adjustments when call frame is not reserved.

## Callee-Save Strategy

### Register-to-Register Spilling
Backend can spill some callee-saved registers into other GPRs instead of stack slots (`assignCalleeSavedSpillSlots`). Register-to-register spill mapping is tracked in `GPRTOCSGPRMap` and used in spill/restore emission.

This optimization reduces stack traffic when caller-saved registers are available to hold callee-saved values.

### FP Handling
If FP is used, FP register is forcibly marked callee-saved in AIE2/AIE2P `determineCalleeSaves()` because prologue/epilogue redefine it post-RA.

### Callee-Saved Register Sets

AIE2 callee-saved: `lr`, `r16`–`r23`, `p6`, `p7`

AIE2P callee-saved: `lr`, `r8`–`r15`, `p6`, `p7`

## Emergency Spill Slots
AIE2P adds conservative emergency scavenging slots in `processFunctionBeforeFrameFinalized()` when estimated stack size exceeds signed 12-bit immediate range assumptions (~4KB).

It reserves spill slots for:
- Pointer register class (`eP`)
- DJ register class (`eDJ`)

This guarantees register scavenger progress for large-stack functions where immediate offsets cannot reach all frame objects.

## Target-Specific SP/FP Adjustment Encodings

### AIE2
- SP adjustments must be multiples of 32 bytes.
- Uses `PADD_sp_imm_pseudo` for immediate SP adjustments.
- Immediate range: ±2^17 (signed 17-bit field, scaled by 32 → effective ±4MB range).
- Falls back to modifier-register sequence for larger pointer adjustments:
  - Load offset into modifier register.
  - Use pointer-add-with-modifier instruction.
- Hard-fails (assertion) on unsupported extreme ranges.

### AIE2P
- SP adjustments must be multiples of 64 bytes.
- Uses `PADDXM_pstm_sp_imm` for immediate SP adjustments.
- Immediate range: ±2^18 (signed 18-bit field, scaled by 64 → effective ±16MB range).
- Uses modifier-register path for general pointer register adjustment.
- Hard-fails (assertion) on unsupported extreme ranges.

### Range Summary

| Target | Pseudo Opcode | Immediate Bits | Scale | Effective Range |
|--------|--------------|----------------|-------|----------------|
| AIE2 | `PADD_sp_imm_pseudo` | 17 signed | ×32 | ±4MB |
| AIE2P | `PADDXM_pstm_sp_imm` | 18 signed | ×64 | ±16MB |

## Key Design Decisions
- Stack model and offset math are encoded centrally in base frame lowering, with ISA-specific immediate encoding delegated to derived classes.
- Aggressive alignment guarantees are enforced with assertions to catch object-ordering and offset bugs early.
- Optional reg-to-reg callee-save spilling is used to reduce stack traffic when safe.
- Upward-growing stack is architectural — the hardware address space and calling convention are built around this model.

## Invariants Downstream Code Depends On
- All SP adjustments respect target stack alignment granularity (32 for AIE2, 64 for AIE2P).
- If FP is active, it must be in saved-register set to preserve correctness across late prologue insertion.
- Spill/restore mapping in `GPRTOCSGPRMap` must remain consistent across prologue/epilogue.
- Emergency scavenging slots on AIE2P are required for large-stack functions (>4KB) to avoid allocator/scavenger failure.
- Stack grows upward — all frame offset calculations assume positive growth direction.
