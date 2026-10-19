# Memory Allocators from Scratch

A 60 Hz frame is 16.7 milliseconds, and one malloc call on a busy heap can spike past a millisecond on its own while a game makes thousands of them per frame. After a level unload the profiler shows 400 MB free, and a 4 MB load still fails. A pointer to a dead enemy now points at a health pickup. Shipping engines fix all three with a different allocator for each kind of lifetime rather than a faster malloc: a bump pointer for per-frame scratch, a pool for fixed-size particles, a buddy or TLSF for the texture heap, a slab for the entity cache, and a generational handle so a freed slot can be reused without a stale pointer slipping through. You'll build every one of them, each with a live demo, and see which ones Unreal, Unity and Naughty Dog's engine use where.
Time
~80 min
Level
Mid to senior engine programmer
Prereqs
You can read C++ at an intermediate level. You've heard of cache lines. The containers tutorial covers what these allocators sit beneath.
Hardware
Some idea that a cache miss is slow and a page fault is much slower
◂ Build a Game Engine
Phase 1 · Memory & concurrency
Next · Job Systems ▸
What you'll be able to do by the end

    Pick an allocator for a workload by answering four questions: what sizes, what lifetime, what worst-case time, and what kind of waste.
    Write a linear, a stack and a pool allocator, and recognize each one's typical bug (a pointer kept past a reset, a pop to the wrong marker, a stale read of a free-list link).
    Predict how much a request wastes in a buddy allocator and in TLSF, and explain why TLSF's worst case stays fixed (at most 160 instructions per malloc in its authors' measurements) however full the heap is.
    Tell internal from external fragmentation, and fix "plenty free, but the big load fails" by separating allocations by lifetime.
    Lay hot data out as structure-of-arrays, and replace raw entity pointers with generational handles that reject a dead entity.


## Sections

- [Why Every Engine Writes Its Own Allocator](01-why-every-engine-writes-its-own-allocator.md)
- [A Short History of Getting Allocation Right](02-a-short-history-of-getting-allocation-right.md)
- [Under the Call to malloc](03-under-the-call-to-malloc.md)
- [The Linear (Bump) Allocator](04-the-linear-bump-allocator.md)
- [The Stack Allocator (Linear with Markers)](05-the-stack-allocator-linear-with-markers.md)
- [The Pool Allocator (Fixed-Size Free List)](06-the-pool-allocator-fixed-size-free-list.md)
- [The Buddy Allocator](07-the-buddy-allocator.md)
- [TLSF: A General-Purpose Allocator with a Bounded Worst Case](08-tlsf-a-general-purpose-allocator-with-a-bounded-worst-case.md)
- [The Slab Allocator](09-the-slab-allocator.md)
- [Fragmentation: Internal, External, and Measured](10-fragmentation-internal-external-and-measured.md)
- [Array-of-Structures vs Structure-of-Arrays](11-array-of-structures-vs-structure-of-arrays.md)
- [Generational Handles: The Right Pointer for a Slot You'll Free and Reuse](12-generational-handles-the-right-pointer-for-a-slot-youll-free-and-reuse.md)
- [Cheat Sheet](13-cheat-sheet.md)
- [How Shipping Engines Stack These Up](14-how-shipping-engines-stack-these-up.md)
- [Pitfalls](15-pitfalls.md)
- [What's Next](16-whats-next.md)
