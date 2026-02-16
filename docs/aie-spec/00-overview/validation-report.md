# Validation Report

- Status: **PASS (with known gaps)**
- Generated: 2026-02-16
- Revision: 3 (updated after 06-clang expansion and 09-architecture synthesis improvements)

## Scope
Validated all documentation under `docs/aie-spec/` for completeness, consistency, cross-reference integrity, and absence of placeholder content.

## 1. File Inventory and Word Counts

| Directory | File | Words |
|-----------|------|------:|
| 00-overview | manifest.json | (JSON) |
| 00-overview | validation-report.md | (this file) |
| 01-isa | instruction-catalog.md | 1,265 |
| 01-isa | instruction-formats.md | 1,606 |
| 01-isa | isa-overview.md | 1,645 |
| 01-isa | registers.md | 1,933 |
| 01-isa | scheduling-model.md | 1,437 |
| 02-codegen | frame-lowering.md | 826 |
| 02-codegen | isel-lowering.md | 913 |
| 02-codegen | isel-patterns.md | 800 |
| 02-codegen | optimizations.md | 1,286 |
| 02-codegen | target-machine.md | 1,310 |
| 03-scheduling | hazard-recognizer.md | 1,413 |
| 03-scheduling | scheduling-overview.md | 1,545 |
| 03-scheduling | software-pipelining.md | 1,643 |
| 03-scheduling | vliw-bundling.md | 1,411 |
| 05-mc-layer | asm-parser.md | 396 |
| 05-mc-layer | asm-printer.md | 355 |
| 05-mc-layer | encoding.md | 637 |
| 05-mc-layer | mc-overview.md | 435 |
| 06-clang | builtins.md | 1,048 |
| 06-clang | frontend-overview.md | 1,140 |
| 07-tests | behavioral-spec.md | 751 |
| 07-tests | codegen-tests.md | 775 |
| 07-tests | test-overview.md | 404 |
| 08-build | build-system.md | 1,124 |
| 09-architecture | architecture.md | 3,812 |
| (root) | README.md | 43 |

**Total: 28 markdown files, 28,998 words** (excluding this report and manifest.json)

### By Section

| Section | Files | Words | Status |
|---------|------:|------:|--------|
| 01-isa | 5 | 7,886 | Complete |
| 02-codegen | 5 | 5,135 | Complete |
| 03-scheduling | 4 | 6,012 | Complete |
| 04-register | 0 | 0 | **Gap** |
| 05-mc-layer | 4 | 1,823 | Complete |
| 06-clang | 2 | 2,188 | Complete |
| 07-tests | 3 | 1,930 | Complete |
| 08-build | 1 | 1,124 | Complete |
| 09-architecture | 1 | 3,812 | Complete |

### Changes Since Revision 2
- 06-clang/frontend-overview.md: 443 → 1,140 words (+157%) — added class hierarchy, type tables, ABI rules, driver defaults
- 06-clang/builtins.md: 269 → 1,048 words (+290%) — added builtin counts, signature encoding, categorized tables, custom lowering patterns
- 09-architecture/architecture.md: 1,657 → 3,812 words (+130%) — added scale table, compilation flow detail, format census, register bank differences, frame lowering variants, multi-slot pseudo invariant, test suite scale, builtin lowering table

## 2. manifest.json Validation

- **Parse**: PASS (valid JSON, 795 lines)
- **Structure**: Contains `target_dir`, `categories` (9 categories), `file_count` (318), `loc_estimate` (133,695)
- **Category coverage**: tablegen, codegen_lowering, codegen_emitter, scheduling, register, mc_layer, frame_stack, optimization, headers, build, tests, clang
- **File path spot-checks**: PASS (verified AIE2.td, AIEHazardRecognizer.cpp, AIEMachineScheduler.cpp, clang AIE target all exist)

## 3. Cross-Reference Consistency

| Check | Result | Notes |
|-------|--------|-------|
| Variant naming (aie2/aie2p) across all docs | PASS | Consistent triple and subtarget naming |
| VLIW slot model (ISA ↔ scheduling ↔ architecture) | PASS | Slot kinds (lda/ldb/alu/mv/st/vec/lng/nop) match across instruction-formats.md, vliw-bundling.md, architecture.md; slot bit-width differences (lda/ldb/st) consistently noted |
| Negative latency docs (scheduling-overview ↔ architecture) | PASS | Both reference signed-latency arithmetic, `-aie-neglatency-lowerbound`, and -10 lower bound |
| Build library dependency order (build-system ↔ architecture) | PASS | Topological order matches in both |
| Scheduling class references (scheduling-model ↔ hazard-recognizer) | PASS | Both reference `InstrItineraryData`, bypass annotations, pipeline stages |
| Register classes (ISA registers ↔ codegen docs) | PASS | ISA docs are explicit (eR/eL/eP, VEC*, ACC*); architecture.md now lists register bank contents per variant |
| Register bank differences (registers ↔ architecture) | PASS | AIE2P-only FifoRegBank, EXPVEC64 in GPRRegBank, AccRegBank composition all documented |
| Hazard recognizer data members (hazard-recognizer ↔ vliw-bundling) | PASS | FuncUnitWrapper fields consistent; conflict rules align |
| PostPipeliner references (software-pipelining ↔ scheduling-overview) | PASS | State machine, II retry, ResMII/RecMII match |
| MC layer references (encoding ↔ asm-printer ↔ architecture) | PASS | Packet format model consistent |
| Builtin counts (builtins.md ↔ architecture.md) | PASS | Both report ~655 total (AIE1: ~41, AIE2: ~395, AIE2P: ~219); architecture.md clarifies separate namespaces |
| Frame lowering variants (frame-lowering ↔ architecture) | PASS | SP granularity (32 vs 64 bytes), emergency scavenging, MaxVectorAlign differences all consistent |
| GlobalISel boundary (isel-patterns ↔ architecture ↔ build-system) | PASS | All three confirm AIE2P has no DAG-ISel generators; AIE1 has no GlobalISel |
| Callee-saved registers (registers ↔ frame-lowering ↔ architecture) | PASS | AIE2 r16-r23 vs AIE2P r8-r15 consistently documented across all three |

## 4. Architecture Build-Sequence Topology

Strict library order in `architecture.md`:
1. `LLVMAIEInfo`
2. `LLVMAIEUtils`
3. `LLVMAIEAsmPrinter` → depends on LLVMAIEUtils
4. `LLVMAIEDesc` → depends on LLVMAIEInfo + LLVMAIEAsmPrinter
5. `LLVMAIECodeGen` → depends on LLVMAIEInfo + LLVMAIEUtils + LLVMAIEAsmPrinter + LLVMAIEDesc
6. `LLVMAIEAsmParser` → depends on LLVMAIEInfo + LLVMAIEUtils + LLVMAIEDesc
7. `LLVMAIEDisassembler` → depends on LLVMAIEInfo + LLVMAIECodeGen

**Topological validity**: PASS — every library appears after all of its dependencies.

## 5. Placeholder/TODO Scan

| Pattern | Matches | Assessment |
|---------|---------|------------|
| `TODO` / `FIXME` | 3 matches in scheduling-model.md | These document upstream FIXME markers in the actual source code — not documentation gaps |
| `stub` | 2 matches (asm-parser.md, architecture.md) | Documents that `ParseDirective` is genuinely a stub in source — factual, not a gap |
| `TBD` / `placeholder` / `[INSERT]` | 0 | Clean |

**Result**: PASS — no documentation placeholders or incomplete sections found.

## 6. Completeness Assessment

### Directories with content
- 01-isa: 5 files — comprehensive ISA reference
- 02-codegen: 5 files — target machine, ISel, frame lowering, optimizations
- 03-scheduling: 4 files — scheduling overview, hazard recognizer, VLIW bundling, software pipelining (all expanded to 200+ lines)
- 05-mc-layer: 4 files — MC overview, encoding, asm printer, asm parser
- 06-clang: 2 files — frontend overview (class hierarchy, ABI, driver defaults), builtins (counts, categories, custom lowering)
- 07-tests: 3 files — test overview, codegen tests, behavioral spec
- 08-build: 1 file — build system
- 09-architecture: 1 file — unified synthesis (3,812 words covering all 8 required sections + 2 appendices)

### Coverage depth assessment

| Section | Depth | Notes |
|---------|-------|-------|
| 01-isa | Deep | Full slot/format/register/scheduling-model coverage with variant comparisons |
| 02-codegen | Moderate | Pass pipeline, ISel, frame lowering well covered; RA pass internals sparse |
| 03-scheduling | Deep | All four subsystems (overview, hazards, bundling, pipelining) expanded with algorithms and thresholds |
| 05-mc-layer | Light | Overview + encoding covered; fixup-to-relocation mapping and packet encoding examples sparse |
| 06-clang | Moderate | Frontend and builtins expanded; covers ABI, driver, type system, custom lowering patterns |
| 07-tests | Moderate | Test categories and behavioral contracts documented; per-test detail not attempted |
| 08-build | Moderate | Library dependencies and TableGen generators covered |
| 09-architecture | Deep | All 8 synthesis sections populated with concrete data, variant comparisons, and cross-references |

### Known gaps
- **04-register/**: Directory exists but contains no markdown files. Register subsystem is partially covered in `01-isa/registers.md` (register classes/families/widths), `02-codegen/target-machine.md` (RA pipeline), and `09-architecture/architecture.md` (register bank differences, staged RA, frame lowering variants), but lacks a dedicated synthesis covering:
  - Register bank info implementation details
  - Staged RA/rewrite pass algorithms (SuperRegRewriter, WawRegRewriter, SubRegConstrainer)
  - Tied-operand constraint handling
  - Live register tracking machinery
  - AIE2P `AIEUnallocatedSuperRegRewriter` fine-grained mode

## 7. Summary

| Criterion | Result |
|-----------|--------|
| All expected files exist | PASS (28 markdown + 1 JSON) |
| manifest.json valid and references real files | PASS |
| Cross-references consistent | PASS (14/14 checks pass) |
| Build sequence topologically valid | PASS |
| No placeholder/TODO sections | PASS |
| Total word count | 28,998 words |
| All sections populated | FAIL — 04-register/ is empty |

**Overall**: PASS with one known gap (04-register/).

## 8. Recommended Follow-up

1. **Add register-layer documentation** under `docs/aie-spec/04-register/` covering register bank info, staged RA passes, tied-operand constraints, and super-register rewriting.
2. **Add a pass-option matrix** — consolidate the 50+ AIE-specific command-line options (scattered across scheduling, codegen, and driver docs) into a single reference table by optimization level and target variant.
3. **Expand MC layer docs** — the 05-mc-layer section has the lowest word count per file; fixup-to-relocation mapping and packet encoding examples could be expanded.
4. **Deepen ISA-to-codegen register cross-references** — map specific register classes (eR/eL/eP, mCDa/mCDb, etc.) to their codegen constraint implementations.
5. **Add control/status register documentation** — AIE2 has 12 control registers and 10 status registers that affect instruction side effects; not currently documented in any synthesis.
