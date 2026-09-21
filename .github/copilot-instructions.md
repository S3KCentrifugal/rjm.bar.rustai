# Repository Instructions

Read and follow the canonical instructions in [`../AGENTS.md`](../AGENTS.md). They apply to every task in this repository, including the strict read-only policy for the external BAR and Recoil repositories.

That file's "Hard constraints" section lists the five rules a design must not violate (no engine patching, no game-archive changes, no scripting layer, generated FFI structs, no panics across the boundary), and its "Facts an agent must not get wrong" section lists what invalidates work if missed. Its "Workspace" and "Navigation" sections describe every sibling project under `C:\bardev`, which are writable and which are strictly read-only, and where to look in each. Consult it before searching for where a change belongs.
