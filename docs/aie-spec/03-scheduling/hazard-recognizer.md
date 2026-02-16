# AIE Hazard Recognizer

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEHazardRecognizer.h`
- `llvm/lib/Target/AIE/AIEHazardRecognizer.cpp`
- `llvm/lib/Target/AIE/aie1/AIEHazardRecognizerPRAS.h` / `.cpp` (legacy AIE1 variant)

## Component Role

`AIEHazardRecognizer` is the central legality oracle for instruction placement. It extends LLVM's `ScheduleHazardRecognizer` with AIE-specific resource, slot, format, and memory conflict modeling. Every instruction placement decision during scheduling passes through this recognizer.

## Class Structure

### Construction Parameters

```cpp
AIEHazardRecognizer(const AIEBaseInstrInfo *TII,
                    const InstrItineraryData *II,
                    AIEAlternateDescriptors &SelectedAlternateDescs,
                    bool IsPreRA,
                    std::optional<unsigned> ScoreboardDepth = std::nullopt)
```

### Key Data Members

| Member | Type | Purpose |
|--------|------|---------|
| `Scoreboard` | `ResourceScoreboard<FuncUnitWrapper>` | Per-cycle resource occupancy tracking |
| `TII` | `const AIEBaseInstrInfo *` | Target instruction info |
| `ItinData` | `const InstrItineraryData *` | Itinerary timing/resource data |
| `SelectedAltDescs` | `AIEAlternateDescriptors &` | Reference to alternate descriptor map |
| `IssueLimit` | `unsigned` (default 1, up to 6) | Max instructions per cycle |
| `ReservedCycles` | `unsigned` | Countdown for blocked scheduling |
| `PipelineDepth` | `int` | Deepest instruction pipeline stage |
| `MaxLatency` | `int` | Maximum operand-to-operand latency |
| `ObjectEnumerator` | `MemoryObjectEnumerator` (mutable) | Pointer base-object tracking |
| `IsPreRA` | `bool` | Pre-RA vs post-RA context flag |
| `FUDepthLimit` | `std::optional<int>` | Pre-RA only: limits itinerary resource depth |
| `IgnoreUnknownSlotSets` | `bool` | Pre-RA only: allows unknown-slot instructions |

## FuncUnitWrapper — Cycle Resource State

### Data Members

```
static const AIEBaseMCFormats *FormatInterface;  // Architecture format constraints
ResourceSet Required;     // StaticBitSet — functional units that must be reserved
ResourceSet Reserved;     // StaticBitSet — alternatively reserved units
SlotBits Slots = 0;       // uint64_t — occupied VLIW slots
SlotBits Conflicts = 0;   // uint64_t — slots causing format conflicts
MemoryBankBits MemoryBanks = 0;      // uint64_t — occupied memory banks
MemoryObjectsBits MemObjectsBits = 0; // uint64_t — pointer object bits (up to 64)
unsigned IssueCount = 0;  // Instructions issued this cycle
```

### conflict() — Complete Conflict Rules

`FuncUnitWrapper::conflict(Other)` returns `true` if ANY of these conditions hold:

| # | Condition | Description |
|---|-----------|-------------|
| 1 | `Slots & Other.Slots` | Same slot occupied |
| 2 | `MemoryBanks & Other.MemoryBanks` | Same memory bank accessed |
| 3 | `MemObjectsBits & Other.MemObjectsBits` | Same pointer object accessed |
| 4 | `Conflicts & Other.Slots` | This instruction's conflict set overlaps other's slots |
| 5 | `Slots & Other.Conflicts` | Other's conflict set overlaps this instruction's slots |
| 6 | `Required.overlap(Other.Required)` | Both require same functional unit |
| 7 | `Reserved.overlap(Other.Required)` | Reserved FU conflicts with other's required FU |
| 8 | `Required.overlap(Other.Reserved)` | Required FU conflicts with other's reserved FU |
| 9 | `Slots && Other.Slots && !isFormatAvailable(Slots | Other.Slots)` | Combined slot set has no valid format |

Rules 1-8 are fast bitwise checks. Rule 9 is the definitive format legality check, only evaluated when both sides have slots.

## ResourceScoreboard

### Template Parameterization

`ResourceScoreboard<FuncUnitWrapper>` is a circular buffer of `FuncUnitWrapper` entries, one per cycle.

### Depth and Window

- Default depth: `max(2 × PipelineDepth, UserScoreboardDepth)`, rounded to power of 2
- Pre-RA depth: from `-aie-scoreboard-depth` (default 128)
- Cycle 0 = current cycle; positive = future; negative = past (for bottom-up scheduling)
- `isInRange(DeltaCycles)` validates cycle index bounds

### How Instructions Are Entered

`enterResources()` maps itinerary stages to scoreboard entries:

1. Create `FuncUnitWrapper` from slot set and conflict set at `Scoreboard[DeltaCycles]`
2. For each itinerary stage:
   - Create `FuncUnitWrapper` from `InstrStage` (Required/Reserved functional units)
   - Write to `Scoreboard[DeltaCycles + CycleOffset]` for each cycle in the stage
3. Add memory bank/object bits at memory access cycles
4. Increment `Scoreboard[DeltaCycles].IssueCount`

## getHazardType() — Complete Algorithm

```
getHazardType(SUnit *SU, int DeltaCycles):
  1. Meta instruction check → NoHazard (IMPLICIT_DEF, KILL bypass all checks)
  2. Reserved cycles check → NoopHazard (unless instruction has delay slot)
  3. Issue limit check → NoopHazard if Scoreboard[DeltaCycles].IssueCount >= IssueLimit
  4. If instruction has alternate opcodes:
     for each AltOpcode:
       if getHazardType(Scoreboard, MI, AltDesc, DeltaCycles) == NoHazard:
         SelectedAltDescs.setAlternateDescriptor(MI, AltOpcode)
         → NoHazard
     → NoopHazard (all alternatives have hazards)
  5. Single descriptor: getHazardType(Scoreboard, MI, DeltaCycles)
```

### DeltaCycles Parameter

- Positive: scheduling into future cycles
- Zero: current cycle
- Negative: backward scheduling (bottom-up)
- Memory access cycles are offset: `AccessCycle = DeltaCycles + Cycles - 1`

### checkConflict()

The inner conflict check examines:
1. **Slot/format conflicts** at the issue cycle (`DeltaCycles`)
2. **Functional unit conflicts** across all itinerary stages
3. **Memory bank/object conflicts** at calculated memory access cycles

Each check calls `FuncUnitWrapper::conflict()` against the corresponding scoreboard entry.

## Memory Hazard Handling

### Memory Bank Conflicts

Memory banks are derived from address spaces:
```
for each memory operand MMO:
  AddrSpace = MMO->getAddrSpace()
  if AddrSpace != 0:
    MemBank = ASI.getMemoryBanksFromAddressSpace(AddrSpace)
  else if aie-addrspace-none-is-safe:
    MemBank = 0  (no conflict)
  else:
    MemBank = ASI.getDefaultMemoryBank()  (conservative)
```

Bank conflicts are checked at memory access cycles (not issue cycles), reflecting when the memory subsystem is actually accessed.

### Pointer Object Tracking

`MemoryObjectEnumerator` assigns bit positions (0-63) to pointer base objects:
- Uses `getUnderlyingObject()` to consolidate derived pointers to the same base
- Parent-child relationships share the same bit position
- When full (64 objects used), returns `nullopt` (conservative fallback)
- Disabled in pre-RA mode to avoid over-constraining early scheduling

Controlled by `-aie-recognize-pointer-hazards` (default true).

### Memory Hazard Design

Memory bank conflicts are modeled conservatively for performance, not correctness:
- The hardware can stall for memory bank conflicts
- The compiler tries to avoid them to prevent stalls
- Pointer object bits provide additional precision beyond bank-level analysis

## Forwarding Path Modeling

Forwarding behavior is modeled via schedule itinerary bypass annotations (e.g., `MOV_Bypass`, `VEC_Bypass`, `MV_Bypass` in AIE2/AIE2P schedule TableGen). These modify effective operand timing by allowing results to be available earlier through bypass paths.

This interacts with signed-latency scheduling: bypass annotations can produce negative effective latencies when the consumer reads from an earlier pipeline stage than the producer writes to.

## Bundle Application and Format Ordering

### applyBundles()

Reconstructs VLIW bundles from collected scheduler output:
1. Remove BUNDLE pseudo-instruction root
2. Remove meta instructions temporarily
3. For multi-instruction bundles: call `applyFormatOrdering()`
4. Re-insert meta instructions after the bundle

### applyFormatOrdering()

Aligns instruction order with the packet format's slot layout:
1. Iterate slots in format order
2. For each occupied slot: remove instruction from current position, re-insert at format-specified position
3. Establish bundle relationships
4. Call `finalizeBundle()` (AIE2/AIE2P only)

## Legacy AIE1 Variant (AIEHazardRecognizerPRAS)

Inherits from `AIEHazardRecognizer`. File is marked "OBSOLESCENT" with intent to delete.

Key differences:
- Maintains local `std::vector<AIE::MachineBundle> Bundles` and `CurrentBundle`
- Uses its own `AIEAlternateDescriptors PRASAlternateDescriptors` (isolated)
- `DeltaCycles == 0` always (no backward scheduling support)
- `AdvanceCycle()` finalizes current bundle and starts a fresh one
- `EmitNoop()` is implemented as `AdvanceCycle()` (creates empty bundle gap)
- `EndBlock()` calls `applyBundles()` to finalize all collected bundles

## Command-Line Options

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-scoreboard-depth` | `128` | Override scoreboard depth |
| `-aie-addrspace-none-is-safe` | `true` | Address space 0 is conflict-free |
| `-aie-recognize-pointer-hazards` | `true` | Enable pointer/base-object hazard tracking |
| `-aie-premisched-fu-depth` | `16` | Pre-RA: functional unit tracking depth limit |
| `-aie-premisched-ignore-unknown-slots` | `true` | Pre-RA: allow unknown-slot instructions |
| `issue-limit` | `6` | Max instructions per cycle |
| `vliw-instrs` | `-1` | Switch off VLIW after N instructions |

## Key Design Decisions

- **Three-layer conflict model**: Functional unit conflicts (traditional), slot/format conflicts (VLIW-specific), and memory bank/object conflicts (AIE-specific) are unified in a single `FuncUnitWrapper::conflict()` check. This avoids separate passes and ensures all constraints are evaluated atomically.
- **Scoreboard-based, not interlock-based**: The compiler constructs a conflict-free schedule; the hardware does not interlock on most hazards. This makes scheduling a correctness requirement, not just an optimization.
- **Pre-RA vs post-RA modes**: Pre-RA uses reduced FU depth and ignores unknown slots for less conservative scheduling. Post-RA requires full precision because the schedule directly determines the emitted instruction stream.
- **Conservative memory modeling**: Memory bank conflicts are for performance (avoiding stalls), while slot/FU conflicts are for correctness. This distinction allows the memory layer to be tuned independently via command-line options.

## Invariants Downstream Code Depends On

- Scoreboard state accurately reflects emitted instructions per cycle.
- Hazard checks use the same descriptor (original or alternate) that is later materialized into final machine instructions.
- Bundle reordering/finalization preserves format legality and scheduler assumptions.
- In post-RA mode, non-meta instructions must resolve to a valid scheduling class; missing sched-class information is a fatal error.
- Memory bank/object conflict checks are disabled in pre-RA mode.
- `applyFormatOrdering()` must run before emission to ensure correct slot order.
