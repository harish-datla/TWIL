# Saving live

An autosave fires mid-fight. If it captures the player's health after this frame's damage but the enemy's attack timer before it, the save describes a moment that never happened. The save must be a consistent snapshot taken at a tick boundary, not mid-update with half the systems stepped. Capture the state at the end of a frame, then write it off-thread so the I/O does not hitch the game. The hard case is streaming: some cells are streamed out, their live objects do not exist in memory, so the snapshot merges the live objects' current state with the last-known persisted delta for the streamed-out regions.

Worked example

Say the authoritative state is 2 MB. Copying it at the end of a frame at roughly 10 GB/s takes about 0.2 ms, which the frame can absorb. Writing those 2 MB to a drive at roughly 100 MB/s takes about 20 ms, more than a whole 60 fps frame. So the copy happens on the game thread at the tick boundary, and the write happens on another thread from the copy.
write on the game thread
copy, then write on a worker
state 2 MB
disk 100 MB/s
autosave now
copy 0.20 mswrite 20.0 ms on the game threadautosave frame 29.2 ms: 1 refresh missed
The game thread runs 9 ms of game work in each 16.7 ms frame, and an autosave fires every 3 seconds (or press autosave now); the model copies at roughly 10 GB/s. With the defaults and the write on the game thread, the autosave frame takes 9 + 0.2 + 20 = 29.2 ms and misses a refresh: a visible hitch. Switch to copy, then write on a worker and the game thread pays only the 0.2 ms copy while the worker lane spends 20 ms writing. Drag the state size to 64 MB and the copy alone costs 6.4 ms, while the write takes 640 ms at 100 MB/s, all of it on the worker.
A state save needs no determinism (unlike lockstep netcode)

This is the clean contrast with the netcode tutorial. Deterministic lockstep ships only inputs, so every machine must compute bitwise-identical results to stay in sync[19]. A save serializes state directly, so loading restores the world wholesale, no replay, no determinism requirement. The one exception is a replay-style save that stores an input log instead of state: that inherits lockstep's exact determinism constraint. State snapshots and command logs are different designs with different failure modes; do not conflate them.
What's intentionally missing

The production save system also needs: the streamed-out-region delta merge in full, save-thread synchronization, encryption and anti-tamper for competitive titles, cloud-save conflict resolution, per-chunk checksums, endianness byte-swapping for cross-platform saves (a PC save loaded on a console), and compaction of dead change-forms so a long playthrough's save does not grow without bound. The Asset Pipeline tutorial owns the byte-level format and endianness machinery this shares.
Check Your UnderstandingQ1 of 5

You serialize an enemy target reference by writing the Player object's raw memory address. On reload the enemy points at garbage. The fix?
Serialize the reference as a stable id or handle, then re-resolve it to the live object on load (two passes: instantiate everything, then fix up). Addresses change every run; ids do not.
Save the Player first so it gets the same address next run.
Use a raw array index instead of a pointer.
0/5 correct

Wrong answers, and why: addresses cannot be pinned across runs and a bare index is fragile, so references are saved as generational handles and fixed up on load; an appended field needs a version branch or a tagged format, and refusing old saves discards progress; a checksum detects torn-write corruption but only an atomic temp-rename prevents it; fsync is about durability, not atomicity; and a change record for a deleted object is skipped, not fatal.
