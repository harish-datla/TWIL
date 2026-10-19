# The Slab Allocator


Spawning an enemy costs more than its bytes. Its constructor sets up a lock, empty containers and default component state, and when the enemy dies and another spawns a moment later, a plain allocator makes you pay for all of that again. Bonwick's 1994 paper makes the same observation about operating-system kernels: a kernel object has to be constructed as well as allocated. A kernel object typically carries locks, condition variables, and reference counts that must be initialized before first use. Free the object and allocate another, and a plain allocator makes you pay for that initialization again. The slab idea is to keep freed objects in their constructed state so the next allocate hands one back ready to use, with the initialization amortized across reuses[6]. The 2001 follow-up added per-CPU magazines so the fast path touches only CPU-local state behind an uncontended per-CPU lock[8]. Linux's SLUB, a descendant design, has been the kernel's only slab allocator since SLAB was removed in 6.8[25].

A slab is a contiguous region of memory carved into objects of one type. A slab cache is a set of slabs, sorted into full slabs (no free objects), partial slabs (some free, some used), and empty slabs (all free, ready to be returned to the page allocator). Bonwick's paper keeps them in one list in that order, so allocation always drains partial slabs before breaking into an empty one. Allocation pops a free object from a partial slab. Free pushes it back; if that leaves the slab with every object free, the slab moves to the empty end of the list and can be reclaimed.

The widget below shows three slabs of a single object type. Each slab is a row of slots, labelled empty / partial / full. The next allocation comes from a partial slab. The magazine column on the right is one CPU's stack of recently freed objects. Only that CPU touches it, so a push or pop needs no shared lock (Solaris guards each CPU's magazines with a per-CPU lock that is almost never contended). An object parked in the magazine is still allocated as far as its slab knows, drawn with a purple outline. The slab layer only gets involved when the magazine is empty on an allocation or full on a free:
Live · Slab cache + per-CPU magazine
ALLOC ◯
FREE ✕
BURST OF 8
RESET ⟲
magazine
0 / 4
magazine hits
0
slab-layer trips
0
empty slabs
3
Each row is a slab of 12 slots: colored slots are in use, purple outlines are parked in the magazine, and dim slots are free; the column on the right is the magazine. Try the worked example below and compare the two counters. The magazine is the fast path. Alternate FREE and ALLOC and every operation bounces off the top of the magazine without touching the slabs, the property Bonwick & Adams added magazines for[8]. The slab layer is involved only when the magazine is empty on an allocation or full on a free. Solaris moves whole magazines to and from a shared depot at that point; this widget moves one object at a time.

Worked example

Start empty, with a magazine that holds 4 objects. Six allocations find the magazine empty, so all six go to the slab layer: 6 slab-layer trips, all from the same partial slab.

Five frees: the first four park in the magazine (4 hits, magazine 4 / 4), and the fifth finds it full and goes back to its slab (7 trips).

Four more allocations pop the four parked objects, still constructed: 8 magazine hits in all. Of the 15 operations, 8 touched only the CPU-local magazine.
Why slabs suit entity caches

An ECS (entity component system: entities are IDs, and each component type's data is stored separately) spawns and despawns enemies, bullets, pickups, particles, audio sources. Each type has a fixed object size. The allocation pattern is bursty (a wave of 30 enemies, then 200 bullets) and locality matters (iterating enemies should be cache-friendly). A slab cache covers each of those:

    Fixed size per type, so the only waste is the unused tail of each slab (SunOS 5.4 kept it under 12.5%) plus, for caches with constructors, one extra word per object so the free-list link doesn't overwrite constructed state[6].
    Contiguous storage, so iterating live objects in a slab is sequential, prefetchable, and free of pointer-chasing. This is an AoS layout at the object level: traversals that touch every field of every object hit the line economy the next section describes.
    Per-CPU magazines, so the hot path touches only CPU-local state, even when one thread frees what another allocated.
    Partial slabs first, so allocations fill partly used slabs and fully free ones can go back to the page allocator when the working set shrinks.

Frykholm's "Custom Memory Allocation in C++"[27] and Reinalter's Molecule engine memory series[34] both include fixed-size pool allocators of this shape, without Bonwick's object caching. An ECS archetype chunk has a similar structure: a fixed-size block holding entities of one component-set shape, where allocation takes the next free slot in a partly filled chunk. The magazine layer is the part archetype storage doesn't usually replicate, since ECS schedulers typically batch structural changes rather than allocating from many threads at once.

[Back to Memory Allocators from Scratch](index.md)
