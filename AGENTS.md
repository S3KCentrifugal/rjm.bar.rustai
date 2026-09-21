# rjm.bar.rustai - Agent Instructions

These instructions apply to the whole repository and are the canonical guidance
for every coding agent (Claude Code, Codex, Copilot, Cursor, Gemini CLI, Aider,
Cline, and any other). Vendor files route here; they must not duplicate or
override this file.

## Project

A **Beyond All Reason Skirmish AI written in Rust**, loaded through Recoil's
stable C interface as `libSkirmishAI.so`.

**Status: proposal. No code exists and none is to be written yet.** Read
[`docs/proposal.md`](docs/proposal.md) before anything else; its evidence is
[`docs/review/engine-game-and-cpp-review.md`](docs/review/engine-game-and-cpp-review.md).

### Hard constraints

Settled. Not open for redesign, and a proposal that violates one is wrong.

1. **No engine patching.** Engine modifications do not survive upstream churn.
   Everything must work against a stock Recoil release. If a capability appears
   to need an engine change, it is out of scope - say so rather than designing
   around it.
2. **No game-archive changes.** Modifying `../Beyond-All-Reason` alters the
   archive checksum and locks out stock clients.
3. **No scripting layer initially.** Policy is Rust. Later script hooks are
   planned, which is why policy lives behind traits (proposal §5) - keep that
   seam sharp.
4. **Struct layouts must match the engine byte for byte.**
   `AIINTERFACE_ABI_VERSION_FAIL` (`aidefines.h`) hashes `sizeof()` over every
   event and command struct plus pointer and arch size. Use `bindgen` against
   the pinned engine headers. **Never hand-write `#[repr(C)]` mirrors** - they
   drift and produce a silently dead AI at match start.
5. **A panic must never cross the FFI boundary.** Every `extern "C"` export
   wraps its body in `catch_unwind`.

### Facts an agent must not get wrong

1. **Execution is synchronous on the simulation thread.** `Game.cpp:1748` ->
   `EngineOutHandler.cpp:128` -> each AI in sequence;
   `SkirmishAIWrapper.cpp:457` profiles but never interrupts. Time spent in
   `handleEvent` is time the simulation waits. Expensive work goes to worker
   threads; the sim thread does dispatch and command emission only.
2. **The engine provides no threat map, no influence map and no terrain
   decomposition.** `Map_*` returns raw grids. Building the derived layer is the
   project, not a detail.
3. **CircuitAI's pathfinder is kept, not replaced.** Its five query types
   (`SINGLE, MULTI, WIDE, COST, LINE`) are threat-weighted and asynchronous; the
   engine offers single-target, distance-only, synchronous pathing and cannot
   express them. The engine call is correct in exactly one place - resource
   cluster distances. See proposal §6.
4. **The game can veto what the engine permits.** `game_initial_spawn.lua:479`
   rejects AI start positions the engine server already accepted. Before
   building on an engine primitive, check whether a BAR gadget filters it for AI
   teams. The engine is necessary but not sufficient evidence.
5. **Chat and map pings are attributed to the hosting player, not the AI**
   (`AICallback.cpp:1401`, with an engine TODO). Identity must be embedded in
   message text.

### Relationship to `../rjm.bar.ai`

`../rjm.bar.ai` holds the founding Intent Record and the original copy of the
engine/game/C++ review. This repository is the implementation project. The two
must not drift: if a fact changes, fix both and say so. Consolidating them is
open decision 5 in the proposal.

## Repository map

| Path | Content | State |
| --- | --- | --- |
| `docs/proposal.md` | the project proposal: architecture, CircuitAI comparison, phasing | written |
| `docs/review/engine-game-and-cpp-review.md` | engine, game and C++ evidence with `file:line` citations | written |
| `src/` | the Rust AI | not started |
| `build.rs` | `bindgen` over the pinned engine headers | not started |
| `CMakeLists.txt` | shim invoking cargo, installing the cdylib | not started |
| `data/AIInfo.lua` | `shortName` **must** equal the install directory name | not started |

## Conventions

Rust work here follows `convention-rust` (naming, `From`/`Into`, newtype IDs,
error handling, documentation) and `convention-testing` (AAA, test naming,
factory builders). Load the skill rather than improvising.

Project-specific additions:

- **The FFI boundary is generated, never hand-written.** See constraint 4.
- **`unsafe` is confined to the `ffi` layer.** Layers above it are safe Rust.
  An `unsafe` block anywhere else needs a comment justifying why the safe
  equivalent does not work.
- **Thread affinity is expressed in types.** If something must run on the sim
  thread, it should be impossible to send it to a worker.
- **Cite engine and game claims by `file:line`** so they can be re-verified
  when upstream moves.

## Workspace

Every project sits side by side under `C:\bardev`. Paths below are relative to
this repository's parent.

| Project | Path | What it is | Write policy |
| --- | --- | --- | --- |
| **rjm.bar.rustai** | `.` | this repository - the Rust Skirmish AI | writable |
| **rjm.bar.ai** | `../rjm.bar.ai` | founding Intent Record and the original review | writable |
| **rjm.bar.host** | `../rjm.bar.host` | containerised match harness that plays AI-vs-AI games in CI - **the test bed for this AI** | writable |
| **rjm.bar.docs** | `../rjm.bar.docs` | central game knowledge base for BAR + Recoil | writable; **game facts only** |
| **rjm.bar.tools** | `../rjm.bar.tools` | Rust `smrt-bar-map`: reads `.sd7` archives, parses SMF, derives terrain facts | writable |
| **rjm.bar.editor** | `../rjm.bar.editor` | Godot BAR map editor | writable |
| **s3k-CircuitAI** | `../s3k-CircuitAI` | the user's CircuitAI/BARb fork (`S3KCentrifugal/CircuitAI`, branch `smrt`) - **the reference implementation this project compares against** | writable |
| **rlcevg-CircuitAI** | `../rlcevg-CircuitAI` | upstream CircuitAI (`rlcevg/CircuitAI`) | read-only by convention |
| **bar-Beyond-All-Reason** | `../bar-Beyond-All-Reason` | upstream BAR game mirror (`beyond-all-reason/Beyond-All-Reason`, `master`) | **strictly read-only** |
| **Beyond-All-Reason** | `../Beyond-All-Reason` | the user's BAR game fork | writable, but see constraint 2 |
| **bar-RecoilEngine** | `../bar-RecoilEngine` | upstream Recoil engine mirror (`beyond-all-reason/RecoilEngine`, `master`) | **strictly read-only** |
| **bar-RecoilEngine-s3k-build** | `../bar-RecoilEngine-s3k-build` | local build clone (origin = `C:\bardev\bar-RecoilEngine`) | build tree; do not author source here |
| Game install | `%LOCALAPPDATA%\Programs\Beyond-All-Reason\data` | engine releases, `AI/Skirmish/BARb/`, logs, replays, 500+ maps | read for evidence; never edit |

### Read-only policy

`bar-Beyond-All-Reason` and `bar-RecoilEngine` are authoritative local
references and are strictly read-only. Never edit, create, delete, rename,
format, generate, build, test, checkout, reset, clean, or update submodules in
either tree. Limit commands there to inspection: file reads, listings,
`git status`, `git log`, `git show`, `git diff`, `git grep`, `git ls-files`. If
a change belongs upstream, report it and name the target file without applying
it. Do not follow a path into a read-only tree and write through it.

Do not assume another machine has this layout; verify paths before use.

## Navigation

Where to look, by repository. Grep is fine - struct names, event topics, unit
ids and gadget filenames are stable tokens.

### `../bar-RecoilEngine` - engine (read-only)

The authority on the ABI and on what the engine permits.

| Path | Content |
| --- | --- |
| `rts/ExternalAI/Interface/SSkirmishAILibrary.h` | the four exported functions a Skirmish AI may provide |
| `rts/ExternalAI/Interface/aidefines.h` | `AIINTERFACE_ABI_VERSION_FAIL` - the struct-layout hash |
| `rts/ExternalAI/Interface/AISEvents.h` | 27 event structs and their ABI sum |
| `rts/ExternalAI/Interface/AISCommands.h` | command topics and structs |
| `rts/ExternalAI/Interface/SSkirmishAICallback.h` | 2,265 lines of accessor: `UnitDef_*`, `WeaponDef_*`, `Map_*`, `Unit_*`, `Economy_*` |
| `AI/Interfaces/C/src/Interface.cpp` | how the interface resolves `init`/`release`/`handleEvent` |
| `rts/ExternalAI/SkirmishAIWrapper.cpp` | how the engine calls the AI; `blockEvents`, `ScopedTimer` |
| `rts/ExternalAI/EngineOutHandler.cpp` | event fan-out to every AI, on the sim thread |
| `rts/ExternalAI/AICallback.cpp` | engine-side implementation of callbacks and legacy commands |
| `rts/Net/GameServer.cpp`, `rts/Net/NetCommands.cpp` | server and client validation of AI-originated messages |
| `rts/build/cmake/Util.cmake` | `get_list_of_submodules` - the glob that discovers an AI directory |
| `AI/Wrappers/Cpp/CMakeLists.txt` | the awk generator `bindgen` replaces |

### `../s3k-CircuitAI` - the reference implementation

Compare against it; do not copy its structure wholesale. Proposal §6 records
what is kept, ported, re-decomposed or deleted.

| Path | Content |
| --- | --- |
| `src/AIExport.cpp` | the 84-line ABI shim this project mirrors |
| `src/circuit/CircuitAI.cpp` | event dispatch as a swappable state machine (`:149`) |
| `src/circuit/scheduler/` | `IMainJob` / `IThreadJob` split and the worker pool |
| `src/circuit/terrain/path/` | **the pathfinder being ported** - `MicroPather`, five query types |
| `src/circuit/map/ThreatMap.h` | per-role, per-domain threat with an atomic double-buffer |
| `src/circuit/map/InfluenceMap.h`, `GridAnalyzer.cpp` | the rest of the derived layer |
| `src/circuit/unit/CircuitDef.h` | caching static defs so the 168 accessors are paid once |
| `src/circuit/task/` | the 114-file task taxonomy being re-encoded |
| `src/circuit/module/` | Economy, Military, Builder, Factory managers |
| `data/script/src/roles/` | the AngelScript role layer being replaced |
| `doc/roles/README.md` | that layer documented, including its defect classes |

### `../rjm.bar.host` - the match harness

Phase 1 plays AI-vs-AI matches in CI and returns a replay plus `result.json`.
`scripts/build-engine-ai.sh` already stages an engine tree and builds an AI
inside it - extend it rather than duplicating it.

### `../rjm.bar.docs` - game truth

Start at `knowledge/README.md`. Layers 10-30 are also queryable from the engine
at runtime; layers 40-80 (roles, counters, tactics, strategy, theory) are
**design-time** inputs with no runtime equivalent. Do not copy game facts into
this repository; name the knowledge-base path instead.

### `../bar-Beyond-All-Reason` - game (read-only)

| Path | Content |
| --- | --- |
| `luarules/gadgets/game_initial_spawn.lua` | the AI start-position veto |
| `common/lib_startpoint_guesser.lua` | what places AI instead |
| `gamedata/modoptions.lua` | mod options visible through `Mod_*` |
| `luarules/gadgets/` | synced game logic that can filter AI actions |

## Source-of-truth routing

Use the narrowest authoritative source:

- AI architecture and implementation: this repository.
- **The ABI and what the engine permits**: `../bar-RecoilEngine`, read-only.
- **What the game permits**: `../bar-Beyond-All-Reason`, read-only.
- **How a problem was solved before**: `../s3k-CircuitAI`, as reference and
  comparison, never as a template to copy.
- Game and unit facts: `../rjm.bar.docs/knowledge/`.
- Rust idiom: the `convention-rust` skill.

## Working rules

- Read the proposal before writing code. If reality contradicts it, fix the
  proposal in the same change - do not let them drift.
- Keep the two read-only trees pristine; inspection commands only.
- Do not commit automatically. Commit only when asked, and only the change
  asked for.
- Cite engine, game and CircuitAI claims by `file:line`.
- Record the BAR and Recoil commits used in document front-matter so staleness
  is detectable.
- When a fact here contradicts `../rjm.bar.docs`, that repository wins for game
  facts and this one wins for AI-implementation facts; fix the other side and
  say so.
