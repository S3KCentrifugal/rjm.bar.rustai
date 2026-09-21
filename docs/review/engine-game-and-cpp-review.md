---
stage: research (pre-design)
status: complete
confidence: data (engine/game/C++ read directly) / derived (implications)
reviewed: 2026-09-21
verified_against:
  - RecoilEngine 92efda5e60 (2026-09-13)
  - Beyond All Reason 1d267c20d1 (2026-09-13)
  - s3k-CircuitAI branch `smrt`
---

# Engine, game and C++ review

Foundations for a Rust Skirmish AI. What the engine actually requires, what the
game actually constrains, what the existing C++ does, and what follows for a
Rust implementation with **no scripting layer**.

This is the evidence beneath [`docs/proposal.md`](../proposal.md) - read the
proposal for what is being built, and this document for why it is shaped that
way. The founding intent is recorded separately in
`../rjm.bar.ai/docs/requirements/intent/intent-record.md`.

## Contents

- [1. What a Skirmish AI is, exactly](#1-what-a-skirmish-ai-is-exactly)
- [2. The execution model](#2-the-execution-model)
- [3. What the engine provides](#3-what-the-engine-provides)
- [4. What the game constrains](#4-what-the-game-constrains)
- [5. The current C++: measured](#5-the-current-c-measured)
- [6. The AngelScript layer and its removal](#6-the-angelscript-layer-and-its-removal)
- [7. Implications for a Rust design](#7-implications-for-a-rust-design)
- [8. Decisions to make](#8-decisions-to-make)

---

## 1. What a Skirmish AI is, exactly

Smaller than it looks. A Skirmish AI is a shared library exporting **four C
functions, of which one is mandatory**.

From `AI/Interfaces/C/src/Interface.cpp:110-175`, which resolves them by name:

```c
enum LevelOfSupport getLevelOfSupportFor(          /* optional */
    const char* aiShortName, const char* aiVersion,
    const char* engineVersionString, int engineVersionNumber,
    const char* aiInterfaceShortName, const char* aiInterfaceVersion);

int init(int skirmishAIId, const struct SSkirmishAICallback* callback);  /* optional */
int release(int skirmishAIId);                                           /* optional */
int handleEvent(int skirmishAIId, int topicId, const void* data);        /* REQUIRED */
```

`init` and `release` are explicitly optional — the source comments say an AI may
use `EVENT_INIT` / `EVENT_RELEASE` instead. Only a missing `handleEvent` calls
`reportInterfaceFunctionError`.

`SSkirmishAILibrary.h` states the rationale: *"The engine will address AIs
through this interface, but AIs will not actually implement it. It is the job of
the AI Interface library to make sure the engine can address AI implementations
through instances of this struct."* The C interface is a thin `dlsym` shim.

The reference implementation in the existing C++ is **84 lines** total
(`s3k-CircuitAI/src/AIExport.cpp`): three exported functions, a
`std::map<int, CCircuitAI*>` keyed by `skirmishAIId`, and a catch-all exception
macro converting throws into error codes.

### The hard constraint: ABI version hashing

`rts/ExternalAI/Interface/aidefines.h`:

```c
#define AIINTERFACE_ABI_VERSION_FAIL ( \
      sizeof(enum LevelOfSupport) \
    + sizeof(struct SSkirmishAILibrary) \
    + sizeof(struct SSkirmishAICallback) \
    + sizeof(struct SAIInterfaceLibrary) \
    + sizeof(struct SAIInterfaceCallback) \
    + AIINTERFACE_EVENTS_ABI_VERSION \
    + AIINTERFACE_COMMANDS_ABI_VERSION \
    + __archBits__   * 10000 \
    + sizeof(int)    * 1001 \
    + sizeof(char)   * 1002 \
    + sizeof(void*)  * 1003 \
```

`AIINTERFACE_EVENTS_ABI_VERSION` and `AIINTERFACE_COMMANDS_ABI_VERSION` are
themselves sums of `sizeof()` over **every** event and command struct
(`AISEvents.h:65+`, `AISCommands.h:169+`).

**Every struct layout must match the engine's byte for byte.** A single padding
difference changes the hash and the interface refuses to load. This is also the
mechanism behind the known failure mode CircuitAI's own README records:

> *"Dead AI upon match start: ensure that `libSkirmishAI.so` is compatible with
> `AI/Interfaces/C/0.1/libAIInterface.so`."*

**Implication for Rust:** hand-written `#[repr(C)]` structs are a liability, not
a shortcut. Generate them with `bindgen` from the engine headers, pinned to the
engine commit, and treat a regenerated diff as a reviewable event. This is not
a style preference — it is the difference between loading and silently dying.

---

## 2. The execution model

**Synchronous, on the simulation thread, unbounded.**

`Game.cpp:1748` calls `eoh->Update()` inside the sim frame.
`EngineOutHandler::Update()` (`EngineOutHandler.cpp:128`) fans out
`DO_FOR_SKIRMISH_AIS(Update(gs->frameNum))` over every active AI in sequence.
Each reaches `CSkirmishAIWrapper::HandleEvent` (`SkirmishAIWrapper.cpp:457`):

```cpp
int CSkirmishAIWrapper::HandleEvent(int topic, const void* data) const {
    ScopedTimer timer(GetTimerNameHash());

    if (!blockEvents || (topic == EVENT_RELEASE))
        return library->HandleEvent(skirmishAIId, topic, data);

    return 0;   // to prevent log error spam, signal: OK
}
```

Three things follow:

1. **There is no timeout and no watchdog.** `ScopedTimer` is profiling only. An
   AI that takes 40 ms in `handleEvent` simply makes the simulation take 40 ms
   longer. With 8 AI instances the cost multiplies.
2. **`blockEvents` exists** as an engine-side mute, so an AI cannot rely on
   receiving every event unconditionally.
3. **Work done inside `handleEvent` is work the simulation waits for.** Any
   expensive computation — pathfinding, threat maps, cluster analysis — must be
   moved off this thread or amortised across frames.

The existing C++ solves this with an explicit scheduler
(`src/circuit/scheduler/`, 542 lines) that distinguishes `IMainJob` (runs on the
sim thread at a given frame) from `IThreadJob` (runs on a worker pool), with
`RunJobAt`, `RunJobAfter`, `RunJobEvery`, `RunParallelJob`, `RunPriorityJob` and
a `WorkerThread(int num)` pool.

**Implication for Rust:** the same split is mandatory, and Rust is better suited
to it. `Send`/`Sync` make the sim-thread/worker boundary a compile-time property
rather than a convention. But the FFI boundary imposes its own rule: **a panic
must never unwind into C**. Every exported function needs
`catch_unwind` at the boundary, mirroring what `CATCH_CPP_AI_EXCEPTION` does for
C++ exceptions in `AIExport.cpp`.

---

## 3. What the engine provides

### Events — 27, push-based, sensor-split

`AISEvents.h`. Perception is delivered, not polled.

| Group | Events |
| --- | --- |
| Lifecycle | `INIT`, `RELEASE`, `UPDATE`, `LOAD`, `SAVE` |
| Own units | `UNIT_CREATED`, `UNIT_FINISHED`, `UNIT_IDLE`, `UNIT_DAMAGED`, `UNIT_DESTROYED`, `UNIT_GIVEN`, `UNIT_CAPTURED`, `UNIT_MOVE_FAILED` |
| Enemies | `ENEMY_CREATED`, `ENEMY_FINISHED`, `ENEMY_DAMAGED`, `ENEMY_DESTROYED`, `ENEMY_ENTER_LOS`, `ENEMY_LEAVE_LOS`, `ENEMY_ENTER_RADAR`, `ENEMY_LEAVE_RADAR` |
| Other | `WEAPON_FIRED`, `SEISMIC_PING`, `COMMAND_FINISHED`, `MESSAGE`, `LUA_MESSAGE`, `PLAYER_COMMAND` |

The enemy group is split by **sensor**, not by existence: `ENTER_LOS` and
`ENTER_RADAR` are distinct events with distinct information content. Fog of war
is handed to the AI as an explicit modelling problem rather than hidden.

Event payloads are small PODs — `SUpdateEvent { int frame; }`,
`SUnitCreatedEvent { int unit; int builder; }`,
`SInitEvent { int skirmishAIId; const SSkirmishAICallback* callback; bool savedGame; }`.

`LOAD` and `SAVE` imply the AI's state should be serialisable. The existing C++
does not meaningfully implement this.

### Callbacks — 2,265 lines of accessor

`SSkirmishAICallback.h`, grouped by prefix:

| Group | Count | Notes |
| --- | --- | --- |
| `UnitDef_*` | 168 | complete static unit data |
| `WeaponDef_*` | 99 | plus `WeaponDef_Damage`, `WeaponDef_Shield` |
| `Map_*` | 42 | raw sensor and terrain grids |
| `Unit_*` | 36 | live state; enemy values LOS-gated |
| `Mod_*` | 32 | mod options and rules |
| `Game_*` | 30 | frame, teams, speed |
| `FeatureDef_*` | 22 | reclaimable features |
| `Economy_*` | 9 | current, income, usage, pull, storage, excess, share, sent, received |
| others | — | `Group_*`, `Cheats_*`, `DataDirs_*`, `File_*`, `Log_*` |

Everything in knowledge-base layers 10–30 is queryable in-process. **A Rust AI
should not ship a unit stat table** — it can read the authoritative values at
runtime.

### Commands

`AISCommands.h`. The full player order set (`UNIT_MOVE`, `UNIT_ATTACK`,
`UNIT_BUILD`, `UNIT_RECLAIM_*`, `UNIT_RESURRECT_*`, `UNIT_CAPTURE_*`,
`UNIT_D_GUN`, `UNIT_GUARD`, `UNIT_FIGHT`, `SET_FIRE_STATE`, `SET_MOVE_STATE`,
`SET_TRAJECTORY`, `WAIT_*`, load/unload), plus:

- **Engine services**: `PATH_INIT`, `PATH_GET_NEXT_WAYPOINT`,
  `PATH_GET_APPROXIMATE_LENGTH`, `PATH_FREE`, `TRACE_RAY`, `TRACE_RAY_FEATURE`.
- **Introspection**: `DEBUG_DRAWER_GRAPH_*` (live plotted lines),
  `DEBUG_DRAWER_OVERLAYTEXTURE_*`, `DRAWER_FIGURE_*`, `DRAWER_POINT_ADD`.
- **Communication**: `SEND_TEXT_MESSAGE`, `DRAWER_POINT_ADD` — both networked,
  both attributed to the hosting player rather than the AI
  (`AICallback.cpp:1401` carries an explicit engine TODO about this).

### What the engine does *not* provide

Worth stating plainly, because it defines the project's real surface area:

- **No threat map.** No influence map. No strategic decomposition of terrain.
- **No clustering** of resource spots beyond raw positions.
- **No notion of a base, a front, or a lane.**

`Map_*` returns raw grids: `getLosMap`, `getRadarMap`, `getJammerMap`,
`getSonarMap`, `getSeismicMap`, `getHeightMap`, `getSlopeMap`, `getSpeedModMap`,
`getResourceMapSpotsPositions`. Every interpretation on top is the AI's own
work — and is where the design effort actually goes.

### API direction of travel

RecoilEngine PR #3352 adds an optional `handleIntent` entry point beside
`handleEvent`, inverting who initiates a call. Optional by design. A new AI
should keep its event dispatch shaped so that adding a second entry point is not
a rewrite.

---

## 4. What the game constrains

The engine defines what is *possible*; BAR's Lua layer defines what is
*permitted*. Three findings matter.

**1. Gadgets can veto AI actions the engine allows.** The clearest case:
`luarules/gadgets/game_initial_spawn.lua:479` rejects start positions from AI
teams outright (`isAiTeam → return false`), then assigns them itself via
`GuessStartSpot` (`common/lib_startpoint_guesser.lua:397`). The engine server
*accepts* the AI's position (`GameServer.cpp:1194`); the game discards it at
`NetCommands.cpp:524`. **A Rust AI inherits this.** No AI-side change can move
its own start position in stock BAR.

**Design rule:** before building a capability on an engine primitive, check
whether a BAR gadget filters it for AI teams. The engine is necessary but not
sufficient evidence.

**2. Modifying the game archive is not an option.** Any change to
`../Beyond-All-Reason` alters the archive checksum and locks out stock clients.
Every capability must work against unmodified BAR.

**3. Game state is reachable but awkward.** `Unit_getRulesParamFloat` /
`getRulesParamString` expose BAR gadget state per unit, and `Mod_*` exposes mod
options. These are the seams through which BAR-specific behaviour becomes
visible.

Game *facts* — unit stats, counters, economy curves, doctrine — live in
`../rjm.bar.docs/knowledge/`. Layers 10–30 are also queryable at runtime; layers
40–80 (roles, counters, tactics, strategy, theory) are **design-time** inputs
with no runtime equivalent. That split is itself a design decision the Rust
project must make deliberately: which knowledge is compiled in, which is derived
at runtime from engine data.

---

## 5. The current C++: measured

### Scale

| Component | Files | Lines |
| --- | ---: | ---: |
| `src/circuit/` — the AI itself | 304 | **60,944** |
| `src/lib/` — vendored dependencies | 238 | ~202,000 |
| `src/AIExport.cpp` — the ABI shim | 1 | 84 |
| `data/script/` — AngelScript policy | 88 | ~10,000 |

The headline number for "CircuitAI is 260k lines" is misleading: **the AI logic
is ~61k lines**, and three quarters of the tree is vendored third-party code.

### Subsystems

| Subsystem | Files | Lines | What it is |
| --- | ---: | ---: | --- |
| `task/` | 114 | 15,140 | the unit-task taxonomy — builder, fighter, static, common |
| `module/` | 12 | 10,573 | Economy, Military, Builder, Factory managers |
| `terrain/` | 31 | 10,204 | blocking map, terrain data, **own pathfinder** |
| `unit/` | 39 | 5,937 | `CircuitDef`, `CircuitUnit`, ally/enemy/action models |
| `util/` | 44 | 4,833 | |
| `map/` | 8 | 3,394 | **ThreatMap, InfluenceMap, GridAnalyzer** |
| `resource/` | 16 | 3,086 | metal/energy spot management |
| `script/` | 21 | 2,987 | AngelScript binding |
| `setup/` | 6 | 1,433 | start position, commander, config |
| `spring/` | 8 | 576 | engine glue |
| `scheduler/` | 3 | 542 | job scheduling, worker threads |

Largest single files: `TerrainManager.cpp` (4,066), `EconomyManager.cpp`
(2,426), `MilitaryManager.cpp` (2,297), `BuilderManager.cpp` (2,003),
`CircuitAI.cpp` (1,855), `FactoryManager.cpp` (1,846), `MicroPather.cpp`
(1,319).

### The dependency stack

| Library | Lines | Used by | Rust equivalent |
| --- | ---: | ---: | --- |
| angelscript | 113,242 | scripting layer | **deleted** |
| lemon | 88,010 | 8 files (graph algorithms) | `petgraph` |
| asbind20 | 14,430 | 1 file (AS binding) | **deleted** |
| json | 8,361 | 19 files (configs) | `serde` / `serde_json` |
| kdtree | 2,084 | 3 files (spatial queries) | `kiddo` |
| triangulate | 592 | 2 files | `spade` / `delaunator` |

**Over half the vendored code — ~128,000 lines — exists solely to host the
scripting layer.** Dropping AngelScript removes it outright.

The C++ *callback wrapper* is not in this tree at all: `AI/Wrappers/Cpp` holds
only 395 checked-in lines and **generates** the object-oriented wrapper with awk
scripts at build time (`AI/Wrappers/Cpp/CMakeLists.txt:200-207`). The Rust
equivalent of that generator is `bindgen`.

### Patterns worth keeping

- **Event dispatch as a swappable state machine.** `CCircuitAI::HandleEvent`
  (`CircuitAI.cpp:149`) is `return (this->*eventHandler)(topic, data);`, with
  `NotifyGameEnd` and `NotifyResign` swapping the handler. This maps cleanly to
  a Rust enum + `match`, with the added benefit that Rust can make illegal
  states unrepresentable.
- **Explicit main/worker job split** (`scheduler/`), given §2.
- **Caching static defs.** `CircuitDef` wraps `UnitDef` so the 168 accessors are
  paid once, not per query. Essential — each callback is an FFI call.
- **Deriving the missing map layers** (`ThreatMap`, `InfluenceMap`,
  `GridAnalyzer`) rather than expecting the engine to supply them.

### Patterns to reconsider

- **Its own pathfinder.** `MicroPather.cpp` (1,319 lines) plus `PathFinder`,
  `PathQuery`, and five query types, while the engine exposes `PATH_INIT` /
  `PATH_GET_NEXT_WAYPOINT` / `PATH_GET_APPROXIMATE_LENGTH`. The engine API is
  used only for `GetApproximateLength` in `MetalManager.cpp:230`; the
  `InitPath` / `GetNextWaypoint` call site in `TerrainManager.cpp:3407` is
  **commented out**. The reason is presumably that cost-map and multi-target
  queries have no engine equivalent — but this is 1,300 lines of A* that should
  be a deliberate decision, not an inherited one.
- **Very large manager files.** Four files over 2,000 lines each, one over
  4,000. Module boundaries that a fresh design can draw better.
- **Config in JSON beside the binary.** 19 files read JSON profiles. Since the
  engine exposes all unit data at runtime, much of what is configured could be
  derived instead.

---

## 6. The AngelScript layer and its removal

Today the C++ is a *mechanism* layer and AngelScript is the *policy* layer.
`data/script/src/roles/` holds six roles — `air`, `front`, `sea`, `support`,
`tactical`, `tech` — totalling 6,102 lines, each registering a `RoleConfig` of
22 delegate slots that the C++ managers call back into.

### What removing it costs

- Policy changes require a recompile rather than a file edit.
- The role abstraction has to be rebuilt in Rust, or replaced by something else.
- Per-profile tuning currently expressed in AngelScript globals
  (`Global::RoleSettings::<Role>`, ~350 values) needs another home.

### What removing it buys

- **~128,000 lines of vendored dependency deleted** (angelscript + asbind20).
- **`src/circuit/script/` — 2,987 lines — deleted.**
- One language, one type system, one debugger. The current split means a bug can
  live in C++, in AngelScript, or in the binding between them.
- Type-checked policy. The AngelScript layer has a documented class of defects
  that are exactly what a type system prevents: handler slots silently unfilled,
  settings declared and never read, copy-pasted role identifiers
  (see `../s3k-CircuitAI/doc/roles/README.md`, "Cross-role findings").
- Compile-time verification that every role implements every required behaviour
  — a trait, rather than 22 nullable function pointers checked at runtime.

### Where policy goes instead

Three viable shapes, to be decided in technical design:

1. **Traits.** A `Role` trait with required methods; each role a type. Closest to
   the current model, fully type-checked, zero runtime cost.
2. **Data-driven with typed config.** Policy as `serde`-deserialised structs
   validated at load. Keeps edit-without-recompile for *values* while keeping
   *behaviour* in Rust.
3. **Derived.** Compute policy from engine data and knowledge-base-encoded
   theory rather than declaring it. Most ambitious, and most aligned with the
   project's stated first-principles intent.

### On later scripting hooks

The stated plan is to add script hooks later. The one thing to preserve now is
**a clean boundary between mechanism and policy** — the same seam AngelScript
occupies today. If policy is expressed behind a trait, a future scripting
backend implements that trait. If policy is scattered through the mechanism
layer, no later hook is cheap. Choosing shape 1 or 2 above keeps that door open
at no present cost; choosing 3 may close it.

---

## 7. Implications for a Rust design

**1. The FFI layer is generated, thin, and pinned.**
`bindgen` over `AISEvents.h`, `AISCommands.h`, `SSkirmishAICallback.h`,
`SSkirmishAILibrary.h`, pinned to an engine commit. A hand-written `#[repr(C)]`
mirror will eventually drift and fail the ABI hash. Wrap in a safe layer that
converts raw pointers and `topicId` into a typed `Event` enum once, at the
boundary.

**2. Four exported functions, all panic-proofed.**

```rust
#[no_mangle]
pub extern "C" fn handleEvent(ai_id: c_int, topic: c_int, data: *const c_void) -> c_int {
    catch_unwind(|| { /* ... */ }).unwrap_or(ERROR_SHIFT + 5)
}
```

A panic crossing into C is undefined behaviour. This mirrors
`CATCH_CPP_AI_EXCEPTION` and is non-negotiable.

**3. Instance registry keyed by `skirmishAIId`.** Multiple AI instances share one
loaded library — the C++ uses `std::map<int, CCircuitAI*>`. In Rust, a
`Mutex<HashMap<i32, Ai>>` or a slab; note §2's single-threaded dispatch means
contention is nil, but `Sync` is still required for a `static`.

**4. Cache every static def once.** Each callback is an FFI call through a
function-pointer table. The 168 `UnitDef_*` accessors should be read once into
owned Rust structs at init, exactly as `CircuitDef` does.

**5. Worker threads for everything expensive.** Threat maps, clustering,
pathfinding. `rayon` for data-parallel passes; a channel back to the sim thread
for results. Rust's `Send`/`Sync` turns the scheduler's runtime discipline into
a compile-time guarantee.

**6. Build inside the engine tree.** The AI is a subdirectory of an engine
checkout, discovered by `file(GLOB */CMakeLists.txt)`
(`rts/build/cmake/Util.cmake:169`) — so a plain directory works, no submodule
wiring needed. A Rust `cdylib` needs a `CMakeLists.txt` shim that invokes
`cargo build` and installs the resulting `libSkirmishAI.so` beside an
`AIInfo.lua` whose `shortName` matches the directory name. The existing
`rjm.bar.host` build script already solves the surrounding problem and should be
reused.

**7. Crate substitutions** replace ~202,000 lines of vendored C++ with six
dependencies — see the table in §5.

**8. The real work is the derived layer.** The engine gives raw grids; threat
maps, influence maps, terrain decomposition and clustering are the AI's to
build, and are where quality comes from. The existing C++ spends 3,394 lines on
`map/` and 10,204 on `terrain/` — that is the honest scale of this part.

---

## 8. Decisions to make

Carried into technical design. None answered here.

1. **Own pathfinder, or engine pathfinder?** The C++ chose its own and left the
   engine calls commented out. Decide deliberately, with the cost-map
   requirement stated.
2. **Policy shape** — traits, typed config, or derived (§6). This also decides
   how cheap later script hooks are.
3. **Which knowledge layers are compiled in versus derived at runtime?** Layers
   10–30 are available from the engine; 40–80 are not.
4. **Role model at all?** The six-role structure is inherited from CircuitAI and
   is not obviously the right decomposition for a first-principles design. The
   intent record explicitly forbids treating it as a template.
5. **How is the AI tested?** `rjm.bar.host` phase 1 already plays AI-vs-AI
   matches in CI. The Rust AI should target that harness from the first commit.
6. **Save/load support.** `EVENT_LOAD` / `EVENT_SAVE` exist; the C++ does not
   meaningfully implement them. Cheap if designed in, expensive if retrofitted.
7. **`handleIntent` (PR #3352).** Not yet merged. Keep dispatch shaped so a
   second entry point is additive.

## Related

- `../../../rjm.bar.ai/docs/requirements/intent/intent-record.md` — project intent and constraints
- `../../../s3k-CircuitAI/doc/roles/README.md` — the AngelScript role layer in detail
- `../../../rjm.bar.host/docs/autohost-design.md` — the match harness this AI should be tested by
- `../../../rjm.bar.docs/knowledge/` — game facts; layers 40–80 are design-time inputs
