# AIE Scheduling Overview

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEMachineScheduler.h`
- `llvm/lib/Target/AIE/AIEMachineScheduler.cpp`
- `llvm/lib/Target/AIE/AIEInterBlockScheduling.h`
- `llvm/lib/Target/AIE/AIEInterBlockScheduling.cpp`

Related files reviewed:
- Hazard/bundle legality: `AIEHazardRecognizer.*`, `AIEAlternateDescriptors.h`, `AIEBundle.h`, `AIEFinalizeBundle.*`, `AIESlot*.*`
- Pipelining/SWP: `AIEBasePipelinerLoopInfo.*`, `AIEPostPipeliner.*`, `AIESWPSolver.*`
- Multi-slot support: `AIEMultiSlotInstrMaterializer.*`
- Schedule/slot models: `AIE2Schedule.td`, `aie2p/AIE2PSchedule.td`, `AIE2Slots.td`, `aie2p/AIE2PSlots.td`

## Why AIE Scheduling Is Different From Upstream LLVM

AIE scheduling cannot be served by upstream LLVM's generic scheduler because it combines constraints absent from typical targets:

| Constraint | Upstream Support | AIE Requirement |
|-----------|-----------------|-----------------|
| Packet/slot legality | Generic VLIW hooks only | Format-dependent bundling with slot-set to packet-format mapping |
| Operand latencies | Non-negative only | Signed latencies including negative values (exposed pipeline) |
| Loop scheduling | Pre-RA MachinePipeliner | Post-RA modulo scheduling with inter-block fixpoint convergence |
| Block-to-block timing | Not modeled | Loop backedge latency/resource convergence with iterative re-scheduling |
| Alternate opcodes | Not modeled | Schedule-time opcode substitution for slot/resource adaptation |
| Memory conflicts | Basic alias analysis | Memory bank and pointer-object conflict integration in hazard checks |

## Class Hierarchy and Overrides

### AIEScheduleDAGMI
Inherits from `ScheduleDAGMI`. Overrides:
- `enterRegion()` / `exitRegion()` — region boundary tracking for inter-block state
- `schedule()` — three-mode dispatch (gathering, pipelining, normal scheduling)
- `releasePred()` — signed-latency readiness computation
- `finalizeSchedule()` — liveness invalidation when negative latencies are active
- `mayAlias()` — custom alias analysis integration

### AIEScheduleDAGMILive
Inherits from `ScheduleDAGMILive`. Overrides:
- `enterRegion()` / `exitRegion()` — pre-RA region tracking
- `getSchedImpl()` — returns the scheduling strategy

### AIEPostRASchedStrategy
Extends `PostGenericScheduler`. Key member: `AIE::InterBlockScheduling InterBlock`.

Overrides and extensions:
- `pickNodeAndCycle()` — custom top-down/bottom-up switching with negative-latency-aware cycle bumping
- `enterMBB()` / `leaveMBB()` — inter-block state management and SWP prologue/epilogue emission
- `tryCandidate()` — extended candidate scoring (delay slots, loop-aware heuristics, resource demand packing)
- `isAvailableNode()` — DeltaCycles support for bottom-up scheduling with negative latencies

### AIEPreRASchedStrategy
Extends `GenericScheduler`. Key members:
- `SUDelayerMap` — tracks SUnits delayed waiting on pressure reducers
- `PSetThresholds` — per-pressure-set thresholds for pressure-aware scheduling
- `AIEAlternateDescriptors SelectedAltDescs` — pre-RA alternate opcode tracking

## The `schedule()` Method — Three Modes

`AIEScheduleDAGMI::schedule()` dispatches based on `SchedulingStage`:

```
switch (BS.FixPoint.Stage) {
  case GatheringRegions:   → return immediately (collecting regions only)
  case Pipelining:         → build DAG, attempt PostPipeliner modulo schedule
  case Scheduling:         → normal ScheduleDAGMI::schedule()
}
```

The `Pipelining` stage builds the full DAG, then delegates to `PostPipeliner::schedule()` with the current II. On success, the block is marked `PipeliningDone`.

## Negative Operand Latencies

### Problem
AIE has an exposed pipeline where register accesses at different pipeline stages create timing relationships that are naturally negative. On AIE2, late-stage register accesses (e.g., E9) can depend on earlier-stage users (e.g., E1), yielding effective latencies around -8 cycles. Upstream LLVM assumes non-negative latencies throughout its readiness computation, which over-constrains scheduling and misses legal instruction placements.

### Implementation

**Signed readiness in `releasePred()`:**
```
int Latency = AllowNegativeLatencies ? PredEdge->getSignedLatency()
                                     : PredEdge->getLatency();
PredSU->BotReadyCycle = std::max(int(PredSU->BotReadyCycle),
                                  int(SU->BotReadyCycle) + Latency);
```

Both operands are cast to `int` for signed arithmetic. This allows `BotReadyCycle` to reflect that a predecessor can legally be placed earlier than its successor.

**Minimum schedulable cycle with lower bound:**
For bottom-up scheduling, `getMinSchedulableCycle()` uses a static lower bound (`-aie-neglatency-lowerbound`, default -10) to avoid traversing the entire predecessor tree:
```
int EarliestPredSchedCycle = int(SU->BotReadyCycle) + NegativeLatencyLowerBound;
return std::max(EarliestPredSchedCycle, 0);
```

The comment notes: "For AIE2 the latest register access is in E9, this can create negative latencies of -8 with E1 accessors. Still, does not hurt to be a bit more conservative with -10."

**DeltaCycles in `isAvailableNode()`:**
Bottom-up node availability iterates `DeltaCycles` from `CurrCycle - BotReadyCycle` down to `-getMaxDeltaCycles()`, allowing instructions to be placed at earlier cycles than their nominal ready cycle.

**Liveness invalidation:**
```
void AIEScheduleDAGMI::finalizeSchedule() {
  if (AllowNegativeLatencies)
    MRI.invalidateLiveness();  // Negative latencies break linear liveness
  ScheduleDAGMI::finalizeSchedule();
}
```

### Controls

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-negative-latencies` | `true` | Enable signed-latency scheduling |
| `-aie-neglatency-lowerbound` | `-10` | Static lower bound for minimum schedulable cycle |

## Inter-Block Scheduling and Fixpoint Algorithm

### SchedulingStage Enum

```
enum class SchedulingStage {
  GatheringRegions,        // Collecting regions in the block
  Scheduling,              // Active scheduling, including iterative loop-aware rounds
  SchedulingNotConverged,  // Fatal: failed to converge
  SchedulingDone,          // Converged schedule (final for non-loop blocks)
  Pipelining,              // Attempting modulo schedule with current II
  PipeliningDone,          // Modulo schedule succeeded (final)
  PipeliningFailed         // All II attempts exhausted (final)
};
```

### State Machine

```
GatheringRegions → Scheduling → SchedulingDone → Pipelining → PipeliningDone
                       ↺                              ↺
                  (re-schedule               (increment II,
                   with adjusted               retry)
                   margins)                      ↓
                                          PipeliningFailed
```

### Fixpoint Convergence (`updateScheduling`)

The fixpoint algorithm iteratively re-schedules loop blocks with increasing safety margins:

1. **Latency phase**: Per-instruction latency margins (`PerMILatencyMargin`) are increased for instructions whose backedge latencies are not satisfied. This biases the scheduler to place those instructions earlier.

2. **Resource phase**: If latency converged but resource conflicts remain across the backedge, the algorithm first tries biasing instruction depth (`PerMIExtraDepth`), then increases global resource margins.

3. **Convergence check**: `latencyConverged()` examines cross-boundary edges between loop bottom and loop top. For each edge crossing the boundary, it verifies that the distance between producer and consumer (measured in bundle heights) satisfies the signed latency requirement.

4. **Pipelining trigger**: After scheduling converges for a single-region loop, if `PostPipeliner::isPostPipelineCandidate()` returns true, the state transitions to `Pipelining` with initial `II = getResMII()`.

### Loop/Epilogue Analysis

When `-aie-loop-epilogue-analysis` is enabled (default), the scheduler also computes NOP requirements between loops and their epilogue blocks:

- `getCyclesToAvoidDataHazards()`: Checks latency edges from loop instructions to epilogue instructions
- `getCyclesToAvoidResourceConflicts()`: Uses a bottom-up scoreboard to detect resource conflicts between the last loop iteration and the epilogue

The maximum of latency and resource NOPs becomes the safety margin pushed to epilogue blocks.

### FixedpointState

```cpp
class FixedpointState {
  SchedulingStage Stage;
  int LatencyMargin = 0;
  SmallMapVector<MachineInstr *, int, 8> PerMILatencyMargin;
  SmallMapVector<MachineInstr *, int, 8> PerMIExtraDepth;
  int ResourceMargin = 0;
  int II = 0;
  int IITries = 0;
  int MaxLatencyExtent = 0;
  int MaxResourceExtent = 0;
  int NumIters = 0;
};
```

### II Management and Retry

- Initial II: `PostPipeliner::getResMII()` (maximum slot count across all slot types)
- Retry: `updatePipelining()` increments II and IITries on each failure
- Limits: `-aie-postpipeliner-maxii` (default 40), `-aie-postpipeliner-maxtry-ii` (default 20)
- On exceeding limits: `PipeliningFailed` (equivalent to `SchedulingDone` but prevents re-entering pipelining)

## Candidate Scoring Extensions

`AIEPostRASchedStrategy::tryCandidate()` extends upstream `PostGenericScheduler` with:

| Heuristic | Priority | Description |
|-----------|----------|-------------|
| Delay slot priority | High | Prioritize instructions with delay slots |
| Top-zone latency | Medium | `tryLatency()` for top-down scheduling |
| Bot path reduction | Medium | Prefer instructions that reduce critical path by depth |
| Loop-aware heuristics | Medium | Consider earliest loop-carried uses (when `-aie-loop-sched-heuristics` enabled) |
| Bot height reduction | Medium | Prefer instructions that reduce schedule height |
| Resource demand packing | Low | Prefer instructions close to `CurrCycle` to improve bundle density |

## Command-Line Options

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-negative-latencies` | `true` | Enable signed-latency scheduling |
| `-aie-neglatency-lowerbound` | `-10` | Lower bound for negative-latency dependencies |
| `-aie-bottomup-cycles` | `max(int)` | Min cycles scheduled bottom-up |
| `-aie-bottomup-delta` | `128` | Max cycles delta for bottom-up scheduling |
| `-aie-reserved-delay-slots` | `0` | Delay slots left empty |
| `-aie-interblock-scoreboard` | `true` | Initialize inter-block scoreboard |
| `-aie-interblock-alignment` | `true` | Allow successor block alignment |
| `-aie-loop-sched-heuristics` | `true` | Loop-specific picking heuristics |
| `-aie-loop-aware` | `true` | Iterative single-block loop scheduling |
| `-aie-loop-epilogue-analysis` | `true` | Loop/epilogue cross-boundary analysis |
| `-aie-loop-aware-expensive-iterations` | `35` | Max fine-grained convergence iterations |
| `-aie-loop-aware-bias-depth` | `true` | Bias depth for hazard avoidance |
| `-aie-postpipeliner-maxii` | `40` | Maximum II for post-RA pipeliner |
| `-aie-postpipeliner-maxtry-ii` | `20` | Maximum II steps to try |
| `-aie-preassign-multi-slot-instr` | `true` | Static materialization of multi-slot pseudos in loops |
| `-aie-prera-cycle-separators` | `false` | Insert CYCLE_SEPARATOR meta instructions |
| `-aie-premisched-finer-rp-tracking` | `true` | Finer register pressure tracking |
| `-aie-premisched-near-critical-regs` | `2` | Free-register threshold for pressure reduction |

## Legacy AIE1 Path

AIE1 uses a distinct legacy path:
- `AIEHazardRecognizerPRAS` — obsolescent PRAS-based hazard recognizer with local bundle management
- Delay-slot filler pass
- Does not use the modern inter-block scheduling or post-RA pipelining architecture

## Key Design Decisions

- Multi-phase scheduling: gather regions, iteratively schedule with fixpoint convergence, then optionally pipeline. This decomposition allows the scheduler to adapt per-instruction margins without global re-analysis.
- Negative latencies are first-class: signed arithmetic throughout readiness, availability, and cycle bumping. Liveness is explicitly invalidated to prevent downstream misuse.
- Inter-block fixpoint is conservative: safety margins are propagated from loops to epilogues, ensuring correctness even when the epilogue schedule is not yet final.
- Post-RA pipelining is integrated into the scheduling fixpoint, not a separate pass. This avoids redundant analysis and allows the pipeliner to share hazard recognizer state.

## Invariants Downstream Code Depends On

- Post-RA scheduling is mandatory for correctness (NOP insertion, bundle legality).
- Bundle legality must correspond to a valid packet format for occupied slots.
- Selected alternate opcode descriptors remain consistent through scheduling and final materialization.
- Pipelined loop transforms preserve minimum trip-count safety and stage correctness.
- Inter-block safety margins must be respected by epilogue scheduling.
- Liveness information is invalid after post-RA scheduling with negative latencies enabled.
