# Pitfalls
Serializing raw pointers
Addresses change per run. Save ids/handles; fix them up on load.
One-pass load
References resolve before their targets exist. Instantiate all, then fix up.
No version tag
Old saves misread after a format change. Stamp a version; migrate.
Refusing old saves
Deletes player progress. Migrate forward instead.
Editing a default after ship
Silently rewrites the meaning of every old save's missing field.
Saving derived state
Bloat plus a second source of truth that drifts. Recompute it.
In-place writes
A crash mid-write corrupts the save. Temp + fsync + rename.
No fallback save
One damaged file loses everything. Keep the previous save and try it when the newest fails.
Trusting counts before checking
A damaged count can ask for billions of objects. Check sizes against the payload before allocating.
Writing on the game thread
A 20 ms write is a dropped frame. Copy at the tick boundary, write on a worker.
Unsynced directory
On POSIX the rename itself can be lost in a power cut. fsync the directory after renaming.
Checksum without atomic write
Detects corruption, does not prevent it. Do both, for different failures.
