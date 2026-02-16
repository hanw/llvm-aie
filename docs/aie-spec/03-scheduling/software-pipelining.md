# AIE Software Pipelining

## Scope
Primary files:
- `llvm/lib/Target/AIE/AIEPostPipeliner.h`
- `llvm/lib/Target/AIE/AIEPostPipeliner.cpp`
- `llvm/lib/Target/AIE/AIEBasePipelinerLoopInfo.h`
- `llvm/lib/Target/AIE/AIEBasePipelinerLoopInfo.cpp`
- `llvm/lib/Target/AIE/AIESWPSolver.h`
- `llvm/lib/Target/AIE/AIESWPSolver.cpp`
- `llvm/lib/Target/AIE/AIEInterBlockScheduling.h` / `.cpp` (pipelining state machine)

## Two-Level Pipelining Strategy

AIE uses both pre-RA and post-RA software pipelining:

| Level | Framework | Entry Point | When Used |
|-------|-----------|-------------|-----------|
| Pre-RA | Upstream `MachinePipeliner` | `AIEBasePipelinerLoopInfo` | Simple loops with ≤3 stages, adequate register pressure |
| Post-RA | Custom `PostPipeliner` | `InterBlockScheduling` state machine | Complex loops deferred by pre-RA, low-II high-stage-count loops |

The pre-RA pipeliner may intentionally defer loops to post-RA when it detects that the post-RA pipeliner can achieve better results. The post-RA pipeliner operates on physically register-assigned code and integrates with the inter-block scheduling fixpoint.

## Pre-RA Loop Assessment (`AIEBasePipelinerLoopInfo`)

### Loop Acceptance

`AIEBasePipelinerLoopInfo` implements `TargetInstrInfo::PipelinerLoopInfo` to validate loops for pre-RA pipelining:

- Supported loop forms: zero-overhead loops, down-counting loops
- Trip count and guard constraints checked
- Instruction eligibility via `shouldIgnoreForPipelining()`
- Register pressure feasibility via `canAllocate()`

### Assessment Enum

Loop rejection reasons are tracked for diagnostics:
- `InvalidLoopControl`, `NoExitCondition`, `ExitAtNonZero`
- `NotACountedLoop`, `NotDownCounting`, `NotRegular`
- `UnboundedLoop`, `TooLowMinTripCount`, `UnsuitableInitVal`
- `PostPipelinerCandidate` — special: loop is suitable but deferred to post-RA

### When Pre-RA Defers to Post-RA (`preferPostPipeliner`)

**A priori check** (before scheduling): Classifies loop by slot statistics. Certain loop classes (e.g., class 1000) are immediately deferred.

**Post-scheduling check** (after `SMSchedule` is computed):
- Requires `MinTripCount > 1` and no II pragma
- Defers if stage count > 3 and II < 11 (`-aie-postpipeliner-cutoff`)
- Defers if any loop-carried latency ≥ II (multi-stage dependencies)
- Does not defer if II ≥ 4 (`-aie-postpipeliner-limit`)

### canAcceptII() — Register Pressure Validation

```
1. Reject if stage count > LoopMaxStageCount (default 3) without II pragma
2. Reject if guard requirements exceed MaxGuards (default 1-2)
3. Reject if register pressure tracking detects infeasible allocation
```

The `canAllocate()` method replays instructions in reverse stage extraction order, tracking liveness through a `RegPressureTracker`, and checks that no pressure set exceeds its register class limit.

### Scheduling Order

`getNodeOrders()` provides alternative scheduling orders:
1. Instructions WITH predecessors (topologically ordered) — scheduled first
2. Instructions WITHOUT predecessors (sources) — deferred to prevent blocking

## Post-RA Modulo Scheduling (`PostPipeliner`)

### Key Data Members

- `AIEHazardRecognizer HR` — cycle-accurate hazard checking
- `ScheduleDAGMI *DAG` — augmented DAG with multiple loop copies
- `ScheduleInfo Info[]` — per-instruction scheduling metadata (Earliest, Latest, Stage, Cycle)
- `ResourceScoreboard<FuncUnitWrapper> Scoreboard` — modulo resource tracking
- `int II` — current initiation interval
- `int NStages` — stage count of successful schedule
- `int RecMII` — recurrence minimum initiation interval
- `int MinTripCount` — from pragma or loop initialization analysis

### Per-Instruction Metadata (NodeInfo)

```
Scheduled, Cycle, ModuloCycle, Stage     — scheduling state
Earliest, Latest                         — feasible window bounds
StaticEarliest, StaticLatest             — pre-computed static bounds
TweakedEarliest, TweakedLatest           — adjusted for iterative refinement
Slots                                    — resource requirements
LastEarliestPusher, LastLatestPusher      — critical path tracking
Ancestors, Offspring                      — transitive closure sets
```

### schedule() Algorithm

1. **Initialize**: Set up scoreboard (`InsertRange + PipelineDepth`), initialize per-instruction metadata
2. **Forward propagation** (`computeForward`): Propagate Earliest times based on latencies
3. **Backward propagation** (`computeBackward`): Propagate Latest times in fixpoint loop
4. **RecMII computation** (`computeRecMII`): Find longest cycle through backedges
5. **Window adjustment**: Adjust Earliest/Latest for resource contention via ancestor/offspring slot counts; apply loop-carried dependence constraints
6. **Feasibility check**: Reject if `II < RecMII`
7. **Heuristic attempts** (`tryApproaches`): Try multiple scheduling strategies
8. **Solver fallback**: If heuristics fail and `II == TargetII`, invoke Z3 solver
9. **Stage validation** (`checkStages`): Verify trip count sufficiency
10. **Extract schedule**: Produce prologue/kernel/epilogue sections

### ResMII Computation

```
int PostPipeliner::getResMII(MachineBasicBlock &LoopBlock) {
  SlotCounts Counts;
  for (auto &MI : LoopBlock)
    Counts += getSlotCounts(MI.getOpcode(), TII);
  return Counts.max();  // Bottleneck slot determines resource MII
}
```

### RecMII Computation

For each instruction K, examines successors that point to iteration K+1 or beyond (backedges). Uses depth-first search with memoization to find the longest path between an ancestor and K. Circuit length = path length + backedge latency. RecMII is the maximum circuit found.

### Minimum Schedule Length

```
int MinLength = II;
for (int K = 0; K < NInstr; K++) {
  while (Info[K].Earliest > Info[K].Latest + MinLength)
    MinLength += II;  // Bump by II until interval fits
}
```

### Heuristic Strategies

11 predefined heuristic configurations, each with multiple priority discriminators:

| Strategy Type | Description |
|--------------|-------------|
| DefaultStrategy | Select instruction with smallest Latest value |
| CheckFixedSchedule | Validate a solver-provided fixed schedule |
| IterCountSlackStrategy | Hybrid top-down/bottom-up targeting SEF-first stage |
| ConfigStrategy | Multi-priority system with configurable discriminators |

ConfigStrategy discriminators include:
- `NodeNum` — topological order
- `Latest` / `Earliest` — feasibility window
- `Critical` — count of predecessors pushed by this instruction
- `Sibling` — siblings already scheduled
- `LCDLatest` — loop-carried dependence constraint
- `DepLength` — schedule deep first
- `Liveness` — minimize live ranges via output dependencies

Each strategy can run multiple times (`-aie-postpipeliner-heuristic-runs`, default 20) for iterative convergence. Strategies can alternate scheduling direction (top-down/bottom-up).

### Stage Validation

After scheduling, `checkStages()` computes `Stage = Cycle / II` for each instruction. If `MinTripCount - (NStages - 1) <= 0`, the schedule requires more iterations than available. The pipeliner attempts to peel side-effect-free instructions from the first stage to reduce NStages.

### Kernel/Prologue/Epilogue Extraction

`visitPipelineSchedule()` extracts three sections:

1. **Prologue** (NPrologueStages sections): Gradually fills the pipeline; each section includes instructions from progressively more stages
2. **Loop kernel** (1 section): Steady-state execution, II cycles per iteration
3. **Epilogue** (NStages - 1 sections): Drains the pipeline after loop exit

## Z3 Solver Integration (`AIESWPSolver`)

### Two Formulations

**Z3BinarySolver** — Binary decision variables:
- Variables: `I[N][S][C]` — boolean, is instruction N in stage S at modulo cycle C?
- Variable count: `NInstr × NumStages × II`
- Constraints: exactly-one per instruction, at-most-one per slot per cycle, latency inequalities

**Z3IntegerSolver** — Integer decision variables:
- Variables: `S[N]` (stage), `C[N]` (modulo cycle) per instruction
- More compact representation but quadratic constraint growth
- Does not yet support side-effect-free stage optimization

### Constraints Generated

| Constraint Type | Description |
|----------------|-------------|
| Instruction | Each instruction scheduled in exactly one (Stage, Cycle) |
| Slot | At most one instruction per slot per modulo cycle |
| Latency | `Cycle[Dst] - Cycle[Src] >= Lat - Dist × II` |
| Loop-carried | Multi-iteration latency handling |
| Memory bank conflicts | Mutual exclusion for bank-sharing instructions |
| SEF stage | Optional: restrict side-effect instructions from early cycles |

### Integration Flow

1. `PostPipeliner::tryApproaches()` exhausts heuristic strategies
2. If heuristics fail and `II == TargetII` (pragma-specified), invoke solver
3. Solver generates and solves constraint model
4. On success, extracted cycle assignments are validated via `CheckFixedSchedule` strategy
5. If validation fails with current NStages, retry with NStages+1 and SEF stage enabled

### Timeout and Determinism

- `-aie-postpipeliner-solver-timeout` (default 2000 ms)
- `-aie-postpipeliner-deterministic-solver` (default true): Estimates solver time via `0.02 × Rows × Columns` before invoking Z3; skips if estimated time exceeds limit
- Non-deterministic mode: Sets Z3 timeout parameter directly

### Z3 Dependency

Conditional compilation with `#if LLVM_WITH_Z3`. When Z3 is not available, `getSolvers()` returns an empty vector and only heuristic scheduling is used.

## II Retry and State Machine

### Pipelining State Transitions in InterBlockScheduling

```
SchedulingDone → Pipelining (with II = ResMII)
                      ↓
             schedule() succeeds?
            /                     \
         yes                       no
          ↓                         ↓
    PipeliningDone          increment II, IITries
                                    ↓
                           limits exceeded?
                          /              \
                        no                yes
                         ↓                 ↓
                     Pipelining     PipeliningFailed
                     (retry)
```

### Trigger Conditions

Pipelining is triggered after scheduling converges for a single-region loop block:
```
if (BS.getRegions().size() == 1) {
  auto &PostSWP = BS.getPostSWP();
  if (PostSWP.isPostPipelineCandidate(*BS.TheBlock)) {
    BS.FixPoint.II = PostSWP.getResMII(*BS.TheBlock);
    BS.FixPoint.IITries = 1;
    → SchedulingStage::Pipelining
  }
}
```

### Trip Count Management

- Pre-RA: `adjustTripCount()` modifies PHI node with `ADD InitReg + (-Adjust × Step)` in preheader
- Post-RA: `updateTripCount()` adjusts by `-(NStages - 1)` after schedule extraction
- `MinTripCount` sourced from pragma (`aie.loop.tripcount.min`), loop initialization analysis, or constant computation

## Command-Line Options

| Option | Default | Purpose |
|--------|---------|---------|
| `-aie-postpipeliner-maxii` | `40` | Maximum II to attempt |
| `-aie-postpipeliner-maxtry-ii` | `20` | Maximum II increment steps |
| `-aie-postpipeliner-heuristic` | `-1` (all) | Select specific heuristic strategy |
| `-aie-postpipeliner-heuristic-runs` | `20` | Convergence runs per heuristic |
| `-aie-postpipeliner-target-ii` | `0` | Force II for solver experiments |
| `-aie-postpipeliner-solver-timeout` | `2000` | Z3 timeout in ms |
| `-aie-postpipeliner-deterministic-solver` | `true` | Estimate time before solving |
| `-aie-pipeliner-track-regpressure` | `true` | Check register pressure in pre-RA |
| `-aie-pipeliner-max-stagecount` | `3` | Max stages before rejection |
| `-aie-pipeliner-max-guards` | `1` | Max prologue guards |
| `-aie-pipeliner-whole-guard` | `true` | Allow guard around whole loop |
| `-aie-postpipeliner-limit` | `4` | II threshold for PostPipeliner preference |
| `-aie-postpipeliner-cutoff` | `11` | II threshold for high stage count deferral |

## Key Design Decisions

- **Two-tier strategy**: Pre-RA handles simple cases with standard infrastructure; post-RA handles complex cases with target-specific knowledge of physical resources and hazards. This avoids over-engineering pre-RA while still achieving good results for hard loops.
- **Heuristic-first, solver-second**: 11 heuristic strategies cover most practical cases efficiently. Z3 solver is only invoked for pragma-specified II targets where exact solutions matter.
- **Integrated with scheduling fixpoint**: Post-RA pipelining is a state in the inter-block scheduling state machine, not a separate pass. This allows sharing hazard recognizer state and seamless retry with increasing II.
- **Side-effect-free stage peeling**: When trip count is insufficient for the full stage count, the pipeliner can extract SEF-only first stages to reduce prologue requirements.

## Invariants Downstream Code Depends On

- Generated pipelined schedule satisfies both RecMII and ResMII constraints.
- Transformed loop preserves min-tripcount correctness (trip count adjusted by NStages-1).
- Stage extraction (prologue/kernel/epilogue) is structurally consistent for later passes.
- When Z3 is unavailable, heuristic-only path must produce correct (if possibly suboptimal) results.
- PostPipeliner only operates on single-region loops within the inter-block scheduling framework.
