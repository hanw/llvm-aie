# AIE VLIW Bundling

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEBundle.h`
- `llvm/lib/Target/AIE/AIEFinalizeBundle.h`
- `llvm/lib/Target/AIE/AIEFinalizeBundle.cpp`
- `llvm/lib/Target/AIE/AIEAlternateDescriptors.h`
- `llvm/lib/Target/AIE/AIEMultiSlotInstrMaterializer.h`
- `llvm/lib/Target/AIE/AIEMultiSlotInstrMaterializer.cpp`
- `llvm/lib/Target/AIE/AIESlotCounts.h` / `.cpp`
- `llvm/lib/Target/AIE/AIESlotUtils.h` / `.cpp`
- `llvm/lib/Target/AIE/AIESlotStatistics.h` / `.cpp`
- `llvm/lib/Target/AIE/MCTargetDesc/AIEMCFormats.h`
- `llvm/lib/Target/AIE/MCTargetDesc/AIEFormat.h`

## Slot Model

AIE instructions are assigned to slot kinds. Each instruction has exactly one slot, and each cycle's instruction bundle must fit a valid VLIW packet format.

### Slot Kinds and Widths

| Slot | AIE2 Width (bits) | AIE2P Width (bits) |
|------|-------------------|-------------------|
| `lda` | 21 | 20 |
| `ldb` | 16 | 17 |
| `alu` | 20 | 20 |
| `mv` | 22 | 22 |
| `st` | 21 | 20 |
| `vec` | 26 | 26 |
| `lng` | 42 | 42 |
| `nop` | 1 | 1 |

Slot metadata and format legality are generated from TableGen (`AIE2Slots.td`, `aie2p/AIE2PSlots.td`) and queried through `AIEBaseMCFormats`.

### MCSlotInfo

Each slot kind has an `MCSlotInfo` descriptor containing:
- `SlotSet` (`SlotBits`) — bitmask of this slot's occupancy
- `ConflictSet` (`SlotBits`) — closure of slot occupancy including exclusions (e.g., XM slot implies both X and M are occupied)
- NOP opcode for this slot

### VLIWFormat

Packet formats are described by `VLIWFormat`:
- `Opcode` — format's bundle opcode
- `Name` — human-readable format name
- `Slots` (SlotKindRange) — ordered list of slot kinds in this format
- `Size` — byte size of the packet
- `SlotSet` (SlotBits) — precomputed bitmask of all slots in the format

`VLIWFormat::covers(SlotBits)` checks whether the format can accommodate a given set of occupied slots.

### PacketFormats

`PacketFormats` wraps the format table and provides:
- `getFormat(SlotBits)` — find minimum-size format covering the given slots
- `getFormatBySize(SlotBits, Size)` — find format of specific size covering the slots

### FormatIterator

`FormatIterator` iterates over all valid formats for a given slot set. It starts at the first format in the table that covers the requested slots and advances through subsequent matches.

## Bundle Representation (`AIE::Bundle<I>`)

### Data Members

```
const AIEBaseMCFormats *FormatInterface;  // Architecture format constraints
SlotBits OccupiedSlots = 0;              // Bitset of occupied slots
std::vector<I *> Instrs;                 // Instructions in insertion order
std::unordered_map<MCSlotKind, I *, MCSlotKind::Hasher> SlotMap;  // Slot → instruction
std::vector<I *> MetaInstrs;             // IMPLICIT_DEF, KILL (stored separately)
I *BundleRoot = nullptr;                 // TargetOpcode::BUNDLE instruction
```

### canAdd() — Legality Checks

`canAdd(InstOpCode)` performs checks in this order:

1. **Empty bundle**: If bundle is empty, any instruction can start it.
2. **Meta instruction**: `IMPLICIT_DEF` and `KILL` bypass format constraints entirely.
3. **Nested BUNDLE**: Can't nest BUNDLE instructions; reject if `BundleRoot` already set.
4. **Standalone check**: If bundle contains a single unsupported instruction, no more can be added.
5. **Format support**: If instruction opcode is not supported by the format interface, reject.
6. **Conflict set check** (fast rejection): Get the instruction's `MCSlotInfo`, compute `ConflictBits`. If `OccupiedSlots & ConflictBits != 0`, reject immediately. This is O(1) and avoids the more expensive format lookup.
7. **Format availability** (definitive check): Compute `NewSlots = OccupiedSlots | SlotInfo->getSlotSet()`, then call `FormatInterface->isFormatAvailable(NewSlots)`. Only accepts if a valid packet format exists for the combined slot set.

### add() — Instruction Insertion

- Meta instructions go to `MetaInstrs` vector
- BUNDLE instruction stored as `BundleRoot`
- Regular instructions: added to `Instrs`, slot computed via `FormatInterface->getSlotKind(OpCode)`, `SlotMap` and `OccupiedSlots` updated

### getFormatOrNull()

Returns the VLIWFormat for the current `OccupiedSlots`:
- Without size constraint: returns minimum-size format via `PacketFormats::getFormat(OccupiedSlots)`
- With size constraint: returns format of specific size via `getFormatBySize(OccupiedSlots, Size)`

## Alternate Descriptor Mechanism

### AIEAlternateDescriptors

A map from `MachineInstr *` to `const MCInstrDesc *`:

```
class AIEAlternateDescriptors {
  MIAltDescsMap AlternateDescs;  // MachineInstr* → MCInstrDesc*
};
```

Key methods:
- `setAlternateDescriptor(MI, AltOpcode)` — records a concrete opcode for an instruction
- `getSelectedDescriptor(MI)` — returns selected descriptor or `nullopt`
- `getDesc(MI)` — returns selected descriptor or falls back to `MI->getDesc()`
- `getOpcode(MI)` — returns selected opcode or falls back to original

### How Alternates Are Selected

During hazard recognition (`getHazardType`), instructions with alternate opcodes are tested in sequence. The first opcode that produces `NoHazard` is recorded via `setAlternateDescriptor()`. This allows the scheduler to adapt instruction encoding to slot/resource pressure without changing the instruction's semantic meaning.

### Multi-Slot Pseudo Instructions

Multi-slot pseudo instructions (MSPs) are instructions that can be placed in any of several slots. Each MSP has an `Alternatives` vector of concrete opcodes (one per legal slot).

Example: A `LOAD_L0_L1_PSEUDO` can materialize to `LOAD_L0` (slot 0) or `LOAD_L1` (slot 1).

## Multi-Slot Instruction Materialization

### AIEMultiSlotInstrMaterializer

Statically assigns concrete slot opcodes to MSPs in single-block loops before pipelining. This is required because pipelining needs deterministic slot occupancy to compute initiation intervals.

### SlotMapping Heuristic

The `SlotMapping` class maps memory banks to slots:

1. **Reuse existing slot**: If the instruction's memory banks overlap with a bank already assigned to a slot, reuse that slot.
2. **Unused load slot**: If no existing assignment matches, pick the first unused load slot.
3. **Least recently used**: If all load slots are used, cycle through them with a round-robin index.

After assignment, `materializeInstr()` replaces the MSP's descriptor with the concrete opcode:
```
auto OpCode = TII->getSlotOpcode(*Slot, MI);
MI.setDesc(TII->get(*OpCode));
```

### Validation

The materializer validates that:
- No memory bank is assigned to multiple slots (`hasUniqueSlotForBank()`)
- Optionally skips if all MSPs resolve to a single slot (no benefit from static assignment)

## Slot Statistics and II Estimation

### SlotCounts

A fixed-size array (max 16 slots) of per-slot cycle counts with arithmetic operations:
- `max()` — maximum count across all slots (bottleneck identification)
- `totals()` — sum of all counts
- `distance(Other)` — L1 (Manhattan) distance between two slot count vectors
- Arithmetic: `+=`, `-=`, `*=` for aggregation

### SlotStatistics

Loop-level slot pressure analysis for II estimation:

```
class SlotStatistics {
  SlotCounts Fixed;   // Fixed instruction slot counts (scaled by Unit=60)
  SlotCounts Free;    // Multi-slot pseudo slot counts (probability-weighted)
  std::vector<MachineInstr *> MSPs;
  std::unordered_map<MachineInstr *, SlotCounts> MSPSlotCounts;
};
```

**Probability scaling**: `Unit = 60` allows fractional slot contributions. Fixed instructions contribute `Unit` per slot. MSPs contribute `Unit / NumAlternatives` per alternative slot, modeling uniform probability across materialization choices.

**Minimum II computation**:
```
int getMinII() const {
  SlotCounts Total = Fixed + Free;
  return Total.max() / Unit;
}
```

This identifies the bottleneck slot and computes how many cycles are needed per iteration to satisfy that slot's demand.

## Bundle Finalization

### AIEFinalizeBundle Pass

Runs after scheduling. Wraps standalone (unbundled) instructions into LLVM bundle form:
- Iterates all instructions in each basic block
- Skips meta instructions, already-bundled instructions, and hardware loop ends
- Calls `finalizeBundle()` for each standalone candidate

### applyBundles() and applyFormatOrdering()

`applyBundles()` (in `AIEHazardRecognizer`) reconstructs VLIW bundles from collected scheduler output:
1. Remove BUNDLE pseudo-instruction root
2. Remove meta instructions temporarily
3. For multi-instruction bundles, call `applyFormatOrdering()`
4. Re-insert meta instructions after the bundle

`applyFormatOrdering()` ensures instructions within a bundle match the format's slot layout:
1. Iterate slots in format order
2. Remove each instruction from its current position
3. Re-insert at the correct position according to format slot order
4. Establish bundle relationships (`bundleWithPred()`)

## Key Design Decisions

- **Slot-set to format mapping**: Legality is a data-driven lookup, not ad hoc pairwise rules. This scales cleanly as formats change across AIE variants.
- **Two-stage conflict checking**: Fast conflict-set rejection (O(1) bitwise AND) before expensive format-table lookup. This matters because `canAdd()` is called per-instruction per-cycle during scheduling.
- **Late alternate descriptor selection**: Improves schedule feasibility by adapting instruction encoding under pressure without changing semantics.
- **Probabilistic II estimation**: `SlotStatistics` with `Unit=60` scaling allows MSPs to contribute fractionally to slot pressure, giving more accurate II lower bounds than worst-case or best-case analysis.
- **Static MSP materialization for loops**: Required before pipelining because modulo scheduling needs deterministic slot occupancy.

## Invariants Downstream Code Depends On

- Every emitted packet corresponds to a valid VLIWFormat.
- No slot is occupied twice in one bundle.
- Conflict sets are a conservative superset of slot exclusions.
- Final opcode materialization matches the descriptor chosen during hazard checks.
- `applyFormatOrdering()` must be called before emission to ensure instruction order matches format slot order.
- MSP materialization, when enabled, must complete before post-RA pipelining attempts.

## What Problems This Solves Beyond Upstream LLVM

Upstream generic VLIW hooks do not provide:
- Slot-set to packet-format legality as a target-owned data model with format iteration
- Late alternate-descriptor selection tied to hazard checks
- Memory-bank and object-level conflict checks integrated into packet formation
- Probabilistic resource estimation for multi-slot pseudo instructions
- Static pre-pipelining slot assignment driven by memory bank analysis
