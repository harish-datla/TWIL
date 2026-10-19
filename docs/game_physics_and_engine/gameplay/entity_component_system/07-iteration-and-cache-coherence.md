# Iteration and cache coherence

The movement system runs on fifty thousand entities, and whether it takes a tenth of a millisecond or several depends almost entirely on where their positions sit in memory. The performance argument for ECS reduces to one claim: iterating flat, typed arrays is faster than pointer-chasing through polymorphic objects. The gap is not algorithmic (both are O(n)); it is entirely in the constant factor, dominated by cache line utilization.

In an OOP hierarchy, each game object is heap-allocated. Iterating "all objects with a physics component" dereferences a pointer per object. Each pointer leads to a different address. The CPU loads a 64-byte cache line for each dereference; if the useful data is 16 bytes (a Position struct), 48 bytes of each line are wasted. Worse, successive objects are rarely adjacent in memory, so the prefetcher cannot guess the next address and each dereference risks missing every cache level. The spread is large: on the Dean and Norvig figures an L1 reference is around 0.5 ns and an L2 reference around 7 ns, while going out to main memory is around 100 ns[6], about 200 times an L1 hit.

In an archetype ECS, all Position values for entities in one archetype are packed in a contiguous float[]. Iterating it is a sequential scan. The hardware prefetcher detects the stride and loads ahead. Every byte of every cache line contains useful data. For a 12-byte Position struct (3 floats), roughly 5 positions fit per 64-byte cache line. At L1 hit latency, the amortized cost per entity is a fraction of a nanosecond. Whole-loop speedups come out smaller than the 200x latency ratio, because out-of-order cores overlap several misses at once and real loops do real work per entity; Albrecht's restructuring bought about 6x[4]. How much depends on working set size and on cache pressure from everything else running.

Worked example

Fifty thousand 12-byte Positions packed in an array are 600,000 bytes, or 9,375 64-byte cache lines, and the prefetcher streams them. The same positions inside fifty thousand heap objects cost a cache line each, 50,000 lines, mostly in random order that the prefetcher can't predict: more than five times the lines, and many of them full misses.
Live · OOP pointer chase vs ECS sequential scan
play
reset
hit time 20 ms (a miss costs 8×)
OOP cache hits
0
OOP cache misses
0
ECS cache hits
0
ECS cache misses
0
Both panels hold the same 64 entities, 8 to a cache line; the readouts count each side's hits and misses. Press play, and slow the hit time to follow it. With this shuffle the OOP side takes 55 misses and 9 hits, the ECS side 8 misses and 56 hits. The OOP side accesses entities in random order (simulating heap-allocated objects scattered across memory); each access to a new cache line is a miss. The ECS side scans sequentially, reusing each line for several entities before advancing. Both visit all 64 entities exactly once, so the only difference is memory layout. A miss stalls that side while the line is fetched, so the sequential (ECS) side finishes first while the pointer-chasing (OOP) side is still grinding through misses. The per-miss stall is set to 8× a hit here so the clip stays watchable; a real DRAM miss runs roughly 50–200× an L1 hit, so the true gap is wider.
Check Your Understanding Q1 of 2

You have 100,000 entities each with a 12-byte Position component stored in an archetype table. The CPU has 64-byte cache lines. Roughly how many cache line loads does a full sequential iteration require?
18,750, the packed array total divided by the 64-byte line size
100,000, because each entity needs its own line loaded from memory
Roughly 1,500 (the CPU prefetches in bursts of 64 lines)
0/2 correct
