# rjm.bar.rustai

A Beyond All Reason Skirmish AI written in Rust, loaded through Recoil's stable
C interface as `libSkirmishAI.so`.

The engine and the game are used **exactly as shipped** — no engine patches, no
game-archive changes. There is no embedded scripting language; policy and
mechanism are both Rust.

Proposal: [`docs/proposal.md`](docs/proposal.md). Evidence:
[`docs/review/engine-game-and-cpp-review.md`](docs/review/engine-game-and-cpp-review.md).
Agent and contributor rules: [`AGENTS.md`](AGENTS.md).

## Status

**Proposal. No code exists.**

| Phase | Deliverable | State |
| --- | --- | --- |
| — | Engine / game / C++ review | Done |
| — | Project proposal | Done, awaiting review |
| 0 | `bindgen` layer, four exports, typed `Event` enum | Not started |
| 1 | Model: cached defs, unit tracking, economy | Not started |
| 2 | Derived: terrain, clusters, threat map | Not started |
| 3 | Pathfinding: five query types, async | Not started |
| 4 | Tasks: build, attack, retreat | Not started |
| 5 | Policy: roles and openings | Not started |

## The shape of it

Five layers. Only layers 1, 2 and 4 run on the simulation thread, and they do
bounded work — the engine calls the AI synchronously and waits.

```
5. policy     roles, openings, doctrine        Rust traits
4. tasks      unit intent                      sim thread
3. derived    threat, influence, terrain       worker threads
2. model      cached defs, tracking, economy   sim thread
1. ffi        generated bindings, safe events  bindgen
```

## Relationship to CircuitAI

`../s3k-CircuitAI` is the reference implementation, not a template. Proposal §6
records every comparable subsystem and whether it is kept, ported, re-decomposed
or deleted.

The headline: its **threat-weighted asynchronous pathfinder is ported, not
replaced** — the engine's pathfinder cannot express cost maps, multi-target
queries or threat weighting. Its AngelScript policy layer is deleted, which also
removes ~128,000 lines of vendored dependency.

## Testing

`../rjm.bar.host` already plays AI-vs-AI matches in CI and returns a replay and
a result. This AI targets that harness from the first commit.

## Related repositories

Siblings under `C:\bardev`.

| Path | What it is |
| --- | --- |
| `../rjm.bar.ai` | founding Intent Record and the original review |
| `../rjm.bar.host` | match harness — the test bed for this AI |
| `../s3k-CircuitAI` | the C++/AngelScript reference implementation |
| `../rjm.bar.docs` | shared BAR and Recoil game knowledge |
| `../bar-RecoilEngine` | engine source; read-only reference |
| `../bar-Beyond-All-Reason` | game source; read-only reference |
