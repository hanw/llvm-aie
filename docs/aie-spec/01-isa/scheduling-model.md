# Scheduling Model (TableGen)

Sources:
- `llvm/lib/Target/AIE/AIE2Schedule.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PSchedule.td`
- `llvm/lib/Target/AIE/aie2p/AIE2PGenSchedule.td`
- `llvm/lib/Target/AIE/aie1/AIE1Schedule.td` (legacy comparison context)

## 1. Architectural Context

From `AIE2Schedule.td` (line 11):

> "AIEngine is an exposed pipeline architecture. As a result, this file must model the processor pipeline with reasonably high fidelity in order to enable the generation of correct code."

Both AIE2 and AIE2P are modeled primarily with `ProcessorItineraries`/`InstrItinClass` + explicit functional resources, not only generic `SchedWrite` abstractions.

This is a correctness requirement, not merely a performance optimization: the hardware does not interlock, so the compiler must produce hazard-free schedules.

## 2. SchedMachineModel Parameters

### AIE2 (`AIE2SchedModel`)

| Parameter | Value | Notes |
|-----------|-------|-------|
| `IssueWidth` | 1000 | Effective bound comes from VLIW format legality, not this parameter |
| `MicroOpBufferSize` | 1000 | Supports scheduler reasoning including negative-latency workflows |
| `Itineraries` | `AIE2Itineraries` | |
| `LoadLatency` | 5 | Default load result latency |
| `MispredictPenalty` | 4 | Branch misprediction cost |
| `HighLatency` | 37 | Threshold for high-latency classification |
| `CompleteModel` | 0 | Model is incomplete (not all ops covered) |
| `PostRAScheduler` | 1 | Post-RA scheduling enabled |

### AIE2P (`AIE2PSchedModel`)

| Parameter | Value | Notes |
|-----------|-------|-------|
| `IssueWidth` | 1000 | Same strategy as AIE2 |
| `MicroOpBufferSize` | 1000 | Same strategy as AIE2 |
| `Itineraries` | `AIE2PItineraries` | |
| `LoadLatency` | 5 | Marked FIXME in source |
| `MispredictPenalty` | 4 | Marked FIXME in source |
| `HighLatency` | 37 | Marked FIXME in source |
| `CompleteModel` | 0 | |
| `PostRAScheduler` | 1 | |

The `IssueWidth = 1000` and `MicroOpBufferSize = 1000` values are deliberately set high. The actual issue constraints come from VLIW composite format legality, not from a generic issue-width limit. These large values prevent LLVM's generic scheduler from applying artificial bottlenecks that would conflict with the VLIW scheduling model.

## 3. Functional Units and Resource Modeling

### AIE2 Functional Units (26 total)

Organized by domain:

#### Structural Conflict Resources

| FuncUnit | Purpose |
|----------|---------|
| `UPPER_SRS` | Upper half of the shift-round-saturate unit |
| `UPS_UNIT` | Upshift unit |
| `PROC_BUS` | Processor bus |
| `STORE_UNIT` | Memory store interface |
| `LOAD_UNIT_A` | Memory load interface A |
| `SEMAPHORE` | Semaphore interface |
| `DONE_UNIT` | DONE interface |
| `EXEC_TRACE_UNIT` | Execution tracing |
| `PART_WORD_STORE` | Part-word store resource (7-cycle operation) |

#### GPR Register Ports

| FuncUnit | Purpose |
|----------|---------|
| `R_RV_PORT` | R register reading (rv port) |
| `RS_WM_PORT` | R/S register writing (wm port), shared with S RegClass |
| `R_WX_PORT` | R register writing (wx port) |
| `R_WA_PORT` | R register writing (wa port) |

#### Address Register Ports

| FuncUnit | Purpose |
|----------|---------|
| `P_RM_PORT` | Pointer/dimension register reading |
| `P_WM_PORT` | Pointer writing (wm port) |
| `M_WM_PORT` | Modifier writing (wm port) |
| `DJ_WM_PORT` | DJ register writing (wm port) |
| `DN_WM_PORT` | DN register writing (wm port) |
| `DC_WM_PORT` | DC register writing (wm port) |

#### Vector Register Ports

| FuncUnit | Purpose |
|----------|---------|
| `W_RS_PORT` | Vector reading (rs port) |
| `W_WA_PORT` | Vector writing (wa port) |
| `W_WM_PORT` | Vector writing (wm port) |

#### Accumulator Register Ports

| FuncUnit | Purpose |
|----------|---------|
| `CM_RM_PORT` | Accumulator reading (rm port) |
| `CM_WM_PORT` | Accumulator writing (wm port) |
| `CM_WA_PORT` | Accumulator writing (wa port) |

#### Utility

| FuncUnit | Purpose |
|----------|---------|
| `EMPTY_FU` | Dummy resource for empty cycles |

Source comment: "Most FuncUnits below correspond to an entry in one of the tables in the 'Hazards and Conflicts' chapter of the ISA."

### AIE2P Functional Units (73 total)

AIE2P's resource model is auto-generated (`AIE2PGenSchedule.td`) and more granular:
- Grouped execution/port resources across ALU, AGU, DMA-like paths, FIFO, control/status subpaths
- Explicit special resources for part-word-store hazards and conflict spacing
- Separate bypass resources: `MV_Bypass`, `VEC_Bypass`

### Bypass Resources

| AIE2 | AIE2P | Purpose |
|------|-------|---------|
| `MOV_Bypass` | `MV_Bypass` | Forwarding through move operations |
| `VEC_Bypass` | `VEC_Bypass` | Forwarding through vector operations |

Bypass tags are used on itinerary entries to distinguish forwarding vs non-forwarding paths.

## 4. Itinerary Classes

### Scale

| Target | Itinerary Classes | FuncUnits |
|--------|------------------|-----------|
| AIE2 | 278 (`II_*`) | 26 |
| AIE2P | 4010 (`II_*`) | 73 |

The 14x increase from AIE2 to AIE2P reflects AIE2P's operand-class-keyed itinerary specialization (`ItineraryRegPairs`), where the same semantic operation has different resource/latency behavior depending on operand register class.

### AIE2 Itinerary Structure

Each itinerary entry uses: `InstrItinData<II_class, [InstrStage specifications], [operand_latencies], [bypass_list]>`

#### Simple ALU Operations (1-cycle, single write port)

```tablegen
InstrItinData<II_ABS, [InstrStage<1, [R_WX_PORT]>], [1, 1, /*def:srCarry*/ 1]>
InstrItinData<II_ADD, [InstrStage<1, [R_WX_PORT]>], [1, 1, 1, /*def:srCarry*/ 1]>
InstrItinData<II_AND, [InstrStage<1, [R_WX_PORT]>], [1, 1, 1, 1]>
```

Pattern: single-cycle reservation of a write port, operand latencies of 1.

#### Complex Multi-Register Operations (address updates)

```tablegen
InstrItinData<II_ADD_NC, [PrefixCycle<P_WM_PORT>, PrefixCycle<M_WM_PORT>,
                          PrefixCycle<DJ_WM_PORT>, PrefixCycle<DN_WM_PORT>,
                          PrefixCycle<DC_WM_PORT>, SimpleCycle<RS_WM_PORT>],
              [1, 1, 1]>
```

Pattern: address-update operations reserve multiple register-file write ports simultaneously, reflecting the hardware's parallel update of pointer, modifier, and dimension state.

#### Memory Operations (multi-stage with hazard avoidance)

```tablegen
InstrItinData<II_LDA,
    [AvoidPartWordStore, EmptyCycles<1>, SimpleCycle<LOAD_UNIT_A>,
     EmptyCycles<4>, PrefixCycle<P_WM_PORT>, ...], ...>
```

Pattern: loads begin with `AvoidPartWordStore` (hazard-avoidance stage), followed by load-unit reservation, then empty pipeline cycles before write-back to register ports. The `PART_WORD_STORE` resource models a 7-cycle conflict window.

#### Semaphore Operations (4-cycle latency)

```tablegen
InstrItinData<II_ACQ, [EmptyCycles<1>, InstrStage<4, [SEMAPHORE]>], [1, 1]>
InstrItinData<II_ACQ_COND, [EmptyCycles<1>, InstrStage<4, [SEMAPHORE]>], [1, 1, 1]>
```

Pattern: semaphore acquire/release have explicit 4-cycle resource reservation. Comment notes: "Semaphore operations have `hasSideEffects=true`, which introduces ordering edges. These edges are detected and have their latencies adjusted using the instruction latencies through a DAG mutator."

#### Control Flow (zero resource reservation)

```tablegen
InstrItinData<II_J, [], [1, 1]>
InstrItinData<II_JL, [], [1, 4]>  // jump-and-link: 4-cycle latency for link register
```

Pattern: branches don't reserve FuncUnits but have operand latencies for result availability.

## 5. Latency Semantics

AIE schedules use two latency layers:

### Operand Latencies

Per-operand read/write timing for RAW/WAR/WAW dependence resolution. Encoded as the operand-cycle list in `InstrItinData`.

### Instruction Latencies

When side effects are considered committed (critical for stores and synchronization ops). Used for ordering edges between side-effecting instructions.

### Stage Modeling Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| `SimpleCycle<FU>` | One-cycle resource occupancy | `SimpleCycle<R_WX_PORT>` |
| `PrefixCycle<FU>` | One-cycle occupancy at start | `PrefixCycle<P_WM_PORT>` |
| `EmptyCycles<n>` | Pipeline gap timing | `EmptyCycles<4>` (load result delay) |
| `AvoidPartWordStore` | Hazard-avoidance stage | Before loads (PART_WORD_STORE conflict) |
| `InstrStage<n, [FU]>` | Explicit n-cycle reservation | `InstrStage<4, [SEMAPHORE]>` |

### Bypass Tags

Bypass tags on itinerary entries model forwarding vs non-forwarding paths:
- `MOV_Bypass` / `MV_Bypass`: forwarding through move operations
- `VEC_Bypass`: forwarding through vector operations
- `NoBypass`: explicit non-forwarding annotation

## 6. AIE2P Operand-Class-Sensitive Scheduling

AIE2P generalizes the itinerary model by generating many variants keyed by operand register class (via `ItineraryRegPairs`). This captures:

- **Per-operand port pressure**: different register classes use different physical ports
- **Conflict behavior**: certain register class combinations create conflicts that don't exist for others
- **Latency variation**: result availability can differ based on source/destination register class

This is why AIE2P has 4010 itinerary classes vs. 278 for AIE2 — the same semantic operation (e.g., vector multiply) may have dozens of itinerary variants depending on which specific vector/accumulator register classes are involved.

## 7. Design Decisions and RISC Contrast

1. **Scheduling is architectural, not merely performance tuning.**
   Itinerary fidelity is needed for correctness (resource/hazard legality), not just throughput improvement. The exposed pipeline contract means the compiler is responsible for all hazard avoidance.

2. **Operand-class-sensitive scheduling is first-class.**
   Especially in AIE2P, register class choice changes itinerary/resource behavior. This is unlike standard RISC targets where scheduling is largely register-class-agnostic.

3. **VLIW legality + itinerary legality both matter.**
   Packet format constraints (slot composites) and pipeline/resource constraints are jointly enforced. A legal slot combination may still violate resource constraints.

4. **Domain-specific hazards are encoded directly.**
   Semaphore/lock behavior (`SEMAPHORE` FuncUnit), part-word memory hazards (`PART_WORD_STORE`), and multidomain register-port conflicts are explicit in the schedule model. These are not generic pipeline hazards but architecture-specific correctness requirements.

5. **Multi-slot pseudo alternatives are scheduling-coupled by design.**
   `*MultiSlotPseudoInstrInfo.td` encodes the invariant that alternative concrete slot materializations must keep equivalent itinerary/latency behavior, so slot choice for packetization does not silently alter dependency timing.

6. **Register port modeling is explicit.**
   Unlike register-file-agnostic RISC scheduling, AIE models individual read/write ports (e.g., `R_WX_PORT` vs `R_WA_PORT`, `W_RS_PORT` vs `W_WM_PORT`) as distinct functional units. Port conflicts are a primary scheduling constraint.
