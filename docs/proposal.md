---
stage: proposal
status: draft for review
confidence: data (engine, game and C++ read directly) / derived (design proposals)
written: 2026-09-21
verified_against:
  - RecoilEngine 92efda5e60 (2026-09-13)
  - Beyond All Reason 1d267c20d1 (2026-09-13)
  - s3k-CircuitAI branch `smrt`
---

# rjm.bar.rustai — project proposal

A Beyond All Reason Skirmish AI written in Rust.

The engine and the game are used **exactly as shipped**. No engine patches, no
game-archive changes. No embedded scripting language, at least initially.

## Contents

- [1. Summary](#1-summary)
- [2. Why Rust, and why now](#2-why-rust-and-why-now)
- [3. Hard constraints](#3-hard-constraints)
- [4. The surface we build against](#4-the-surface-we-build-against)
- [5. Proposed architecture](#5-proposed-architecture)
- [6. CircuitAI feature comparison](#6-circuitai-feature-comparison)
- [7. Dependency stack](#7-dependency-stack)
- [8. Build and integration](#8-build-and-integration)
- [9. Testing](#9-testing)
- [10. Phasing](#10-phasing)
- [11. Open decisions](#11-open-decisions)

---

## 1. Summary

Build a Skirmish AI for Beyond All Reason in Rust, loaded through Recoil's
stable C interface as `libSkirmishAI.so`.

The AI is a single Rust `cdylib` exporting four C functions. Policy and
mechanism are both Rust; there is no second language in the system. Expensive
computation — threat maps, influence maps, pathfinding — runs on worker threads,
with the simulation thread doing only dispatch and command emission.

The design borrows CircuitAI's proven *mechanisms* where they are good — its
threat-weighted pathfinder in particular — and departs from it where a type
system, a single language, or a fresh decomposition does better. Section 6
records that comparison feature by feature, with source references, so the
divergences are deliberate and reviewable.

## 2. Why Rust, and why now

**One language, one type system.** CircuitAI today is C++ mechanism plus
AngelScript policy plus a binding layer between them. A defect can live in any
of the three. The AngelScript layer has a documented class of failures that are
precisely what a type system prevents: handler slots silently left unfilled,
settings declared and never read, role identifiers copy-pasted between files
(`../s3k-CircuitAI/doc/roles/README.md`, "Cross-role findings").

**Concurrency as a compile-time property.** The AI runs on the simulation
thread and blocks it (§4). Moving work off that thread is structural, not an
optimisation. CircuitAI does this with a hand-rolled scheduler and careful
discipline; Rust's `Send`/`Sync` make the same split checkable by the compiler.

**A large deletion.** Dropping the scripting layer removes ~128,000 lines of
vendored C++ (AngelScript + asbind20) plus CircuitAI's 2,987-line binding
layer, before a line of AI logic is written.

**Unoccupied ground.** Rust bindings to the Spring C AI interface exist
(`spring-ai-sys`, `spring-ai-rs` v0.2.1) but are undocumented and thinly
maintained, with no mention of Recoil or BAR. Useful as reference, not as a
dependency.

## 3. Hard constraints

These are settled, not open for design.

1. **No engine patching.** Engine modifications do not survive upstream churn
   and cannot be relied on. Everything must work against a stock Recoil release.
2. **No game-archive changes.** Modifying `../Beyond-All-Reason` alters the
   archive checksum and locks out stock clients. The AI works against
   unmodified BAR or not at all.
3. **No scripting layer initially.** Policy is Rust. A scripting backend may be
   added later; §5 keeps that seam open at no present cost.
4. **Struct layouts must match the engine byte for byte.** See §4.
5. **A panic must never cross the FFI boundary.**

### What these rule out

Two capabilities investigated and found unreachable:

| Want | Blocked by | Consequence |
| --- | --- | --- |
| AI chooses its own start position | `game_initial_spawn.lua:479` rejects positions from AI teams (`isAiTeam → return false`), then assigns via `GuessStartSpot` | The AI takes the position it is given. Post-spawn relocation is the only lever. |
| Per-AI attribution on chat and map pings | `AICallback.cpp:1401` uses `gu->myPlayerNum`, with an engine TODO saying `team` should be used | Identity must be embedded in message text. |

**Standing design rule:** before building on an engine primitive, check whether
a BAR gadget filters it for AI teams. The engine is necessary but not sufficient
evidence.

## 4. The surface we build against

Condensed from [the full review](review/engine-game-and-cpp-review.md).

**The ABI is four functions, one mandatory.** `AI/Interfaces/C/src/Interface.cpp:110-175`
resolves `getLevelOfSupportFor`, `init`, `release` and `handleEvent` by `dlsym`.
Only `handleEvent` is required. CircuitAI's whole ABI shim is 84 lines.

**Struct layouts are hashed.** `aidefines.h` computes
`AIINTERFACE_ABI_VERSION_FAIL` from `sizeof()` over every event struct, every
command struct, `__archBits__` and pointer size. One padding difference and the
library will not load — the "dead AI at match start" failure CircuitAI's README
records. **This makes `bindgen` mandatory rather than stylistic.**

**Execution is synchronous on the sim thread.** `Game.cpp:1748` calls
`eoh->Update()`; `EngineOutHandler.cpp:128` fans out to every AI in sequence;
`SkirmishAIWrapper.cpp:457` wraps the call in a `ScopedTimer` that profiles but
never interrupts. There is no watchdog. Time spent in `handleEvent` is time the
simulation waits, multiplied by the number of AI instances.

**The engine gives raw material, not interpretation.** 27 events (enemy
lifecycle split by sensor), 168 `UnitDef_*` and 99 `WeaponDef_*` accessors, 42
`Map_*` grid accessors, 9 `Economy_*`, the full player command set, an engine
pathfinder, `TRACE_RAY`, and a debug drawer with live graphs.

It provides **no threat map, no influence map, and no strategic decomposition of
terrain**. Building those is the project.

## 5. Proposed architecture

Five layers, bottom to top. Each depends only on those beneath it.

```
┌──────────────────────────────────────────────────────────┐
│ 5. policy      what to do — roles, openings, doctrine    │  Rust traits
├──────────────────────────────────────────────────────────┤
│ 4. tasks       unit intent — build, attack, retreat      │  sim thread
├──────────────────────────────────────────────────────────┤
│ 3. derived     threat, influence, terrain, clusters      │  worker threads
├──────────────────────────────────────────────────────────┤
│ 2. model       cached defs, unit/enemy tracking, economy │  sim thread
├──────────────────────────────────────────────────────────┤
│ 1. ffi         generated bindings + safe event/command   │  bindgen
└──────────────────────────────────────────────────────────┘
```

**Layer 1 — `ffi`.** `bindgen` output pinned to an engine commit, plus a safe
wrapper converting `(topicId, *const c_void)` into a typed `Event` enum once, at
the boundary. Four `#[no_mangle] extern "C"` functions, each wrapping its body
in `catch_unwind`. Instance registry keyed by `skirmishAIId`.

**Layer 2 — `model`.** Static defs read once at init into owned Rust structs
(each callback is an FFI call through a function-pointer table; paying 168 of
them per query is not viable). Live tracking of own units, allies and enemies,
with LOS-gating made explicit in the types: an enemy's health is
`Option<f32>`, not a stale `f32`.

**Layer 3 — `derived`.** Threat map, influence map, terrain decomposition,
resource clustering, pathfinding. All of it on worker threads, published to the
sim thread as immutable snapshots. This is where the AI's quality lives and
where most of the effort goes.

**Layer 4 — `tasks`.** What a unit is currently trying to do, and the state
machine that advances it. Consumes layer 3 snapshots; emits engine commands.

**Layer 5 — `policy`.** Role behaviour, opening selection, army composition,
when to attack. Expressed as traits so that the mechanism/policy seam stays
sharp — the same seam AngelScript occupies in CircuitAI today. **A future
scripting backend implements those traits.** Keeping policy behind a trait is
what makes later script hooks cheap; scattering policy through layers 2–4 is
what would make them impossible.

### The sim-thread contract

Only layers 1, 2 and 4 run on the simulation thread, and they do bounded work:
dispatch an event, update tracking, advance task state machines, emit commands.
Layer 3 runs on workers and publishes results by atomic pointer swap — the same
pattern CircuitAI uses (§6), which Rust expresses as `arc-swap` with no unsafe
code.

## 6. CircuitAI feature comparison

Notes on each comparable subsystem: what CircuitAI does, where it lives, and how
this project differs. Line counts are `src/circuit/` on branch `smrt`.

| Subsystem | CircuitAI | Lines | Rust plan |
| --- | --- | ---: | --- |
| ABI shim | `src/AIExport.cpp` | 84 | **Same shape.** Four exports, instance map, catch-all. `catch_unwind` replaces the exception macro. |
| Callback wrapper | generated by awk into `src-generated/` | ~200k generated | **`bindgen`.** Same idea, standard tooling, regenerated against a pinned engine commit. |
| Scheduler | `scheduler/` | 542 | **Keep the split, change the mechanism.** `IMainJob`/`IThreadJob` becomes `Send` bounds plus `rayon`; the compiler enforces what discipline enforced. |
| Pathfinding | `terrain/path/` (`MicroPather` 1,319) | ~3,500 | **Port, do not replace.** See below. |
| Threat map | `map/ThreatMap.cpp` | 866 | **Port the design.** Atomic double-buffer becomes `arc-swap`. |
| Influence map | `map/InfluenceMap.cpp` | ~400 | Port; fold into the same snapshot publication. |
| Grid analysis | `map/GridAnalyzer.cpp` | 859 | Port. |
| Terrain | `terrain/` (`TerrainManager` 4,066) | 10,204 | **Re-decompose.** Function preserved, the 4,066-line file is not. |
| Unit model | `unit/` (`CircuitDef`, `CircuitUnit`) | 5,937 | **Port the caching idea, sharpen the types.** LOS-gating becomes `Option`. |
| Economy | `module/EconomyManager.cpp` | 2,426 | Re-decompose; behaviour is a policy concern, accounting is not. |
| Military | `module/MilitaryManager.cpp` | 2,297 | Split mechanism from policy; policy moves to layer 5. |
| Builder / Factory | `module/{Builder,Factory}Manager.cpp` | 3,849 | As above. |
| Tasks | `task/` — 114 files | 15,140 | **Keep the taxonomy, change the encoding.** Class hierarchy → enum + trait. |
| Resources | `resource/` (`MetalManager`) | 3,086 | Port. Retains the one engine-pathfinder call (below). |
| Setup | `setup/` | 1,433 | Port, minus the start-position picker, which the game vetoes. |
| Scripting | `script/` + AngelScript | 2,987 + ~128k vendored | **Deleted.** |

### Pathfinding — port, do not replace

**This is the most important "keep".** The earlier review questioned whether
CircuitAI's own pathfinder should be replaced by the engine's
`PATH_INIT`/`PATH_GET_NEXT_WAYPOINT`. It should not. The API surface explains
why (`terrain/path/PathFinder.h`):

```cpp
std::shared_ptr<IPathQuery> CreatePathSingleQuery(CCircuitUnit*, CThreatMap*, ...);
std::shared_ptr<IPathQuery> CreatePathMultiQuery (CCircuitUnit*, CThreatMap*, ...);
std::shared_ptr<IPathQuery> CreatePathWideQuery  (CCircuitUnit*, const CCircuitDef*, ...);
std::shared_ptr<IPathQuery> CreateCostMapQuery   (CCircuitUnit*, CThreatMap*, ...);
std::shared_ptr<IPathQuery> CreateLineMapQuery   (CCircuitUnit*, CThreatMap*, ...);
void RunQuery(CScheduler*, const std::shared_ptr<IPathQuery>&, PathCallback&& onComplete);
```

Five query types (`PathQuery.h:19`: `SINGLE, MULTI, WIDE, COST, LINE`), every
one of them **threat-weighted** — they take a `CThreatMap*`, so route cost
includes danger, not just distance. And `RunQuery` is **asynchronous**, executed
on the scheduler with a completion callback.

The engine offers single-source, single-target, distance-only, synchronous
pathing. It cannot express cost maps, multi-target queries, wide-formation
paths, or threat weighting. The one place the engine pathfinder genuinely fits
is `MetalManager.cpp:230` — `GetApproximateLength` between resource spots for
cluster distances — and CircuitAI uses it there, correctly. The commented-out
`InitPath`/`GetNextWaypoint` block at `TerrainManager.cpp:3407` records an
experiment that was abandoned.

**Rust plan:** port `MicroPather` and the five query types faithfully. Keep the
async-query-with-callback shape; in Rust the callback becomes a channel send or
a future, and `Send` bounds make the worker-thread handoff checked. Keep the
engine call for resource-cluster distances. Treat the multi-resolution
coordinate helpers (`MoveXY`/`PathXY`/`PathIndex`) as a correctness-critical
port — these are easy to get subtly wrong and cheap to unit-test.

### Threat map — port the design, simplify the concurrency

`map/ThreatMap.h` publishes through an atomic pointer:

```cpp
float* GetAirThreatArray (CCircuitDef::RoleT type) { return pThreatData.load()->roleThreatPtrs[type]->airThreat.data(); }
float* GetSurfThreatArray(CCircuitDef::RoleT type) { ... }
float* GetAmphThreatArray(CCircuitDef::RoleT type) { ... }
float* GetSwimThreatArray(CCircuitDef::RoleT type) { ... }
void EnqueueUpdate();
bool IsUpdating() const;
```

`pThreatData.load()` is a lock-free read of a double-buffered snapshot while a
worker rebuilds the other buffer. Threat is stratified per movement domain (air,
surface, amphibious, swimming) **and per role** — a raider and a bomber see
different threat surfaces over the same ground. That is a genuinely good design
and the reason CircuitAI's pathing behaves well.

**Rust plan:** same structure, `ArcSwap<ThreatSnapshot>` instead of a raw
atomic pointer. Readers get an `Arc` with no unsafe and no lifetime puzzles;
the rebuilding worker constructs a fresh snapshot and swaps it in. The
per-role/per-domain stratification is preserved exactly.

### Tasks — keep the taxonomy, change the encoding

CircuitAI has 114 task files: `builder/` (BigGun, Bunker, Combat, Convert,
Defence, Energy, Factory, Generic, …), `fighter/` (AirWave, AntiAir, AntiHeavy,
Artillery, Attack, Bomb, Defend, Ferry, Guard, …), `static/`, `common/`, plus
`IdleTask`, `NilTask`, `PlayerTask`, `RetreatTask`, `UnitTask`.

The taxonomy is sound and hard-won. The *encoding* — a deep virtual class
hierarchy across 114 files — is where Rust does better: an enum of task kinds
with a shared trait for advancement makes exhaustive matching a compiler
obligation and removes the `NilTask`-style null objects entirely.

### Event dispatch — keep the state machine

`CircuitAI.cpp:149` is `return (this->*eventHandler)(topic, data);`, with
`NotifyGameEnd` and `NotifyResign` swapping the handler pointer. A clean idea:
game-over and resigned states genuinely want different event handling.

**Rust plan:** the same state machine as an enum, so the transitions are
explicit and the compiler rejects unhandled states. This also leaves room for
PR #3352's `handleIntent` as an additional entry point rather than a rewrite.

### Configuration — reconsider

19 files read JSON behaviour profiles, and the AngelScript layer holds roughly
350 tunables in `Global::RoleSettings::*`. Much of this describes things the
engine can already tell us at runtime.

**Rust plan:** typed `serde` config for what genuinely must be tuned, derived
values for what can be computed. Config that duplicates engine data is a
staleness bug waiting to happen.

## 7. Dependency stack

Replacing ~202,000 lines of vendored C++:

| CircuitAI | Lines | Rust crate |
| --- | ---: | --- |
| angelscript | 113,242 | — (deleted) |
| lemon (graphs) | 88,010 | `petgraph` |
| asbind20 | 14,430 | — (deleted) |
| json | 8,361 | `serde`, `serde_json` |
| kdtree | 2,084 | `kiddo` |
| triangulate | 592 | `spade` |
| awk wrapper generator | ~200k generated | `bindgen` |

Plus `arc-swap` for snapshot publication and `rayon` for data-parallel passes.

## 8. Build and integration

The AI builds as a subdirectory of an engine checkout. Discovery is a glob, not
git: `AI/Skirmish/CMakeLists.txt` calls `get_list_of_submodules`, which is
`file(GLOB */CMakeLists.txt)` (`rts/build/cmake/Util.cmake:169`), so a plain
directory is found and no submodule wiring is needed.

Required shape:

```
AI/Skirmish/<Name>/
    CMakeLists.txt          # shim: invoke cargo, install the cdylib
    data/AIInfo.lua         # shortName MUST equal <Name>
    <cargo project>
→ installs  libSkirmishAI.so  beside  AI/Interfaces/C/0.1/libAIInterface.so
```

The directory name is the identity — it becomes the CMake target, the install
path, and must match `shortName` in `AIInfo.lua`.

`bindgen` runs in `build.rs` against the engine headers at the pinned commit.
Moving that pin is an engine upgrade and must be a reviewed diff: it silently
invalidates the interface pairing.

`../rjm.bar.host/scripts/build-engine-ai.sh` already solves the surrounding
problem — staging the engine source, dropping an AI in at `AI/Skirmish/<name>/`,
and building both in Recoil's pinned container. Extend it rather than duplicate
it.

## 9. Testing

Three levels.

**Unit tests** on the pure parts — coordinate conversions, threat accumulation,
cost-map arithmetic, task state transitions. These are the parts most likely to
be subtly wrong and cheapest to pin down.

**ABI smoke test.** Build, load, run one match, assert the AI reached
`EVENT_INIT` and issued commands. The failure this catches — a silently dead AI
from a struct-layout mismatch — is otherwise invisible until match start.

**Match harness.** `../rjm.bar.host` phase 1 already plays AI-vs-AI matches in
CI and returns a replay plus `result.json`. **Target it from the first commit.**
An AI that is not continuously played is an AI whose regressions are discovered
by hand.

## 10. Phasing

| Phase | Deliverable | Done when |
| --- | --- | --- |
| 0 | `bindgen` layer, four exports, typed `Event` enum | loads in a real match, logs events, exits cleanly |
| 1 | Model layer: cached defs, unit/enemy tracking, economy | reports accurate state over a full match |
| 2 | Derived layer: terrain, clusters, threat map | threat surface renders correctly via the debug drawer |
| 3 | Pathfinding: the five query types, async | units move sensibly under threat weighting |
| 4 | Tasks: build, attack, retreat | plays a complete game without stalling |
| 5 | Policy: roles and openings | beats a fixed baseline in the harness |

Phase 0 is the real risk (ABI correctness). Everything after is incremental and
measurable in the harness.

## 11. Open decisions

1. **Policy shape** — traits, typed config, or derived from knowledge. Decides
   how cheap later script hooks are. §5 assumes traits.
2. **Role model.** CircuitAI's six roles are inherited, not derived. The intent
   record forbids treating them as a template; what replaces them is undecided.
3. **Which knowledge layers compile in versus derive at runtime.** Layers 10–30
   are queryable from the engine; 40–80 are design-time only.
4. **Save/load.** `EVENT_LOAD`/`EVENT_SAVE` exist and CircuitAI does not
   meaningfully implement them. Cheap designed in, expensive retrofitted.
5. **Relationship to `../rjm.bar.ai`.** That repository holds the Intent Record
   and the full engine/game/C++ review. Either it is folded into this project or
   this one is its implementation phase — but the two should not drift.

## Related

- [`docs/review/engine-game-and-cpp-review.md`](review/engine-game-and-cpp-review.md) — the evidence beneath this proposal
- `../rjm.bar.ai/docs/requirements/intent/intent-record.md` — project intent and constraints
- `../s3k-CircuitAI/doc/roles/README.md` — the AngelScript role layer being replaced
- `../rjm.bar.host/docs/autohost-design.md` — the match harness this AI is tested by
- `../rjm.bar.docs/knowledge/` — game facts; layers 40–80 are design-time inputs
