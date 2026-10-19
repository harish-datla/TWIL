# Pitfalls

"Cooked means compressed"
Cooking is transform + layout; compression is a separate packaging step.
Crash on ARM, fine on x86
A misaligned cast of a cooked blob: undefined behavior everywhere, and a fault on the ARM cores and instructions that require alignment.
Old files won't load
Bumped the version but wrote no loader for the old layout or default-fill rule.
Reference breaks on rename
Referencing by path instead of a stable GUID.
Stale asset after reload
Holding a raw index/pointer instead of a generational handle.
Cooker re-cooks everything
No dependency graph, or no visited-set so a reference cycle loops.
Rust struct serializes wrong
#[repr(Rust)] field order is unspecified; use #[repr(C)].
zero-copy garbage on another platform
The layout is the target's; re-cook per platform (endian/align/format).

[Back to Asset Pipeline & Serialization](index.md)
