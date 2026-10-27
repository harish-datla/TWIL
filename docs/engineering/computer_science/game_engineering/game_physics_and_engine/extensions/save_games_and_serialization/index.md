# Save Games & Serialization

Save data is mutable player state that must survive format changes, crashes, and interrupted writes. This tutorial covers object references, recomputation, full and delta saves, versioning, migration, atomic writes, integrity checks, and hitch-free autosaves.

Time: ~80 min  
Level: Senior  
Prerequisites: The Gameplay Layer, ECS, and Asset Pipeline.  
Stack: C++ & Rust  
Track: Extensions beyond the capstone

## What you’ll be able to do

- Save object references as IDs or generational handles and reconnect them in two passes.
- Decide what to save, what to recompute, and how to represent world deltas.
- Version save formats and migrate old saves safely.
- Write atomically with temporary files, `fsync`, rename, backups, and CRC-32 checks.
- Capture consistent autosaves at tick boundaries without hitching the frame.

## Sections

- [A Save Is Mutable State, Not Assets](01-a-save-is-mutable-state-not-assets.md)
- [The Object-Graph Problem](02-the-object-graph-problem.md)
- [Save vs Recompute](03-save-vs-recompute.md)
- [Full vs Delta Saves](04-full-vs-delta-saves.md)
- [Versioning & Migration](05-versioning-and-migration.md)
- [Atomic Writes](06-atomic-writes.md)
- [Saving Live](07-saving-live.md)
- [Pitfalls](08-pitfalls.md)
- [What’s Next](09-whats-next.md)
