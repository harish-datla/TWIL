# Fragmentation: Internal, External, and Measured


After a level unload the memory profiler shows 412 MB free on a 1 GB budget, and loading a 4 MB texture still fails. The memory is there; it's in the wrong shape. Fragmentation is the gap between "bytes free" and "bytes usefully allocatable". There are two kinds, and allocator designs mostly trade one for the other. Internal fragmentation is space wasted inside a block: a 65-byte request in a 128-byte buddy slot wastes 63 bytes; a 12-byte object in a 96-byte pool slot wastes 84. External fragmentation is space wasted between blocks: a free-list allocator with thousands of 16-byte holes and a request for 4 KB has plenty of bytes free but no usable block. Pools have only internal fragmentation by construction; linear allocators have neither (until they fail entirely); buddy has bounded internal fragmentation plus external fragmentation whenever a live block keeps its buddy from merging; general-purpose malloc can develop arbitrary external fragmentation.

Worked example

A 4 KB heap holds 64 blocks of 64 bytes, back to back: 64 × 64 = 4,096 bytes, full.

Free every other block. Now 2,048 bytes are free, half the heap, but every free block sits between two live ones, so the largest hole is 64 bytes. A 128-byte request fails.

Free the other half instead, the 32 blocks in the upper 2 KB, and the same 2,048 free bytes form one 2 KB hole. Same amount free, opposite outcome: what matters is where the survivors sit.

The widget below runs the same workload through four allocators in parallel, each on its own 4 KB heap, in two phases. Phase 1 churns: bimodal request sizes, random frees, live bytes held near 60% of the heap, and a slider-set share of allocations marked long-lived and sprinkled through the run. Phase 2 is a level unload: every short-lived block is freed and only the long-lived ones stay. The stat to watch is the largest contiguous free block once phase 2 finishes. With the long-lived mix at zero, every row coalesces back to one 4 KB run. Raise it and each survivor that landed mid-heap splits the free space around it. At the default setting, first fit usually keeps the largest hole, because it packs early allocations toward low addresses, with the TLSF-style segregated fit a little behind; at high mixes the lead changes from run to run. Buddy often ends smallest, because one survivor keeps its whole power-of-two neighborhood from merging. The pool row always reports one slot, since any free slot serves any request:
Live · Fragmentation race · same workload, four allocators
churn ops per run 5000
long-lived mix 0.20
RUN ▶
RESET ⟲
free list largest
4.0 KB
buddy largest
4.0 KB
TLSF largest
4.0 KB
pool largest
32 B
Each strip is one allocator's 4 KB heap: colored segments are live blocks, gaps are free runs, and long-lived blocks have a white cap. Set the long-lived mix to 0 and press RUN: every row but the pool ends with one 4.0 KB hole. Put it back to 0.20 and run a few times: the largest holes drop to roughly 1 to 3 KB. A simulator, not a benchmark of any specific implementation. The "long-lived mix" slider sets how many allocations survive the level unload, the pattern Dubé reports behind 10 to 30 MB of fragmentation on an Xbox 360 title, where level-load temporaries shared pages with permanent allocations[36]: a pinned block in the wrong place caps the largest hole no matter how well the allocator coalesces. Runs are randomized, so the same settings can end with visibly different numbers from one run to the next. One simplification to know about: the pool row treats every request as one fixed 32-byte slot, where a real engine would run one pool per size class.
The worst case is theoretical; the typical case is measured

Robson 1971 proved that any non-relocating allocator has a worst-case overhead bound of Θ(M log n), where M is the maximum simultaneous live bytes and n is the ratio of largest to smallest allocation[12]. The proof constructs an adversarial sequence of allocations and frees that no clever policy can survive without paying that bound. It's why moving garbage collectors can guarantee what a native heap can't.

Wilson, Johnstone, Neely, and Boles 1995 argued from the other direction: allocators should be judged on traces of real C and C++ programs, with the lifetime patterns those programs actually produce, rather than on synthetic random traces[1]. Johnstone and Wilson's 1998 follow-up did the measuring[14]: on eight real programs, best fit and address-ordered first fit averaged under 1% true fragmentation once header and alignment overhead were counted separately. Their explanation is phase behavior: objects allocated at about the same time tend to die at about the same time, and those two policies place such objects next to each other, so their holes merge back into large free blocks.

The two results don't conflict. The theory says any non-moving allocator can be fragmented by an adversarial request stream; the measurements say a good policy (best fit or address-ordered first fit) keeps real programs near zero, while poor ones (next fit, simple segregated storage without coalescing) waste far more. When a shipping engine does fragment, a common cause is a lifetime mix the policy can't see: a long-lived allocation (texture pool, level state, asset cache) lands among short-lived ones and blocks coalescing around it, which is the pattern in the widget above. The usual fix is structural, separating allocations by lifetime or size, more often than a better general-purpose allocator. Bonwick makes the same argument for slabs[6], and MacDougall's GDC 2016 system splits the address space by allocation size for the same reason[35].
Published fragmentation numbers from shipping games are scarce

What exists is mostly GDC slides and developer blog posts. Dubé's blog series reports 10 to 30 MB of fragmentation on an unnamed Xbox 360 title, caused by level-load temporaries sharing pages with permanent allocations[36]. MacDougall's 2016 talk for SCE London Studio describes the fragmentation that pushed the team to a new system and the system itself, without before-and-after numbers[35]; Gyrling's Naughty Dog talk puts the waste from worst-case-sized linear allocators at 100 to 200 MiB[24]. Treat a specific fragmentation figure for a named game with no talk or post behind it as folklore.
Check Your Understanding Q1 of 1

Your engine’s memory profiler shows 412 MB free on a 1 GB budget, but a 4 MB asset load fails. The diagnosis is:
The profiler is misreading reserved-but-uncommitted address space as free memory, and the true free number is near zero.
External fragmentation: the free space is split across many holes, none 4 MB contiguous. Give large allocations a segregated heap, or pick a coalescing policy.
Internal fragmentation: the 412 MB is slack trapped inside allocated pool slots that are bigger than their payloads.
0/1 correct

[Back to Memory Allocators from Scratch](index.md)
