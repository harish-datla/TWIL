# Why Every Engine Writes Its Own Allocator


Your profiler shows a frame that ran 3 ms over budget, and the extra time sits in a handful of malloc calls that each took hundreds of microseconds, while thousands of others in the same frame took about 30 nanoseconds. Nothing in your code changed between the fast calls and the slow ones. The allocator just took its slow path.

A generic malloc has a hard job. It must serve any size, from a 12-byte node to a 16-megabyte texture, with no idea how long the caller will hold it, from any thread, and with no help from the caller about which of those it's about to do next. The price of that generality is a free-list search, a heap-lock acquisition, sometimes a syscall, and an answer whose worst case can be hundreds of times slower than its median. An engine knows more than that. It knows the per-frame scratch lifetime is one frame. It knows particles are 96 bytes each. It knows the texture heap wants 64 KB alignment. The job of an engine allocator is to turn that domain knowledge into faster code with simpler invariants, without ever surprising the frame budget.

The widget below draws allocation latencies from models of two allocators. The first models a general-purpose malloc on a fragmented heap. The second models a linear bump allocator. Both medians sit in the tens of nanoseconds. The difference is the tail: the modeled malloc has a heavy right tail where the rarest calls hit a page fault, a free-list scan, or a heap-lock wait, and its worst sample lands several orders of magnitude above its median. The bump allocator has effectively no tail because every call runs the same handful of instructions.

Worked example

A frame makes 2,000 allocations. In the widget's malloc model at its default fragmentation, the median call takes about 29 ns and the 99th percentile (the time 1 call in 100 exceeds) about 115 µs.

The slow 1%: 20 calls, each at least 115 µs, so at least 20 × 115 µs = 2.3 ms, about 14% of a 16.7 ms frame.

The other 1,980 calls at around 30 ns each: about 60 µs. Twenty calls cost more than 35 times as much as the other 1,980 put together. That's why the tail, not the median, is what an engine allocator has to fix.
Live · Allocator latency distribution
malloc
bump
both
simulated heap fragmentation 0.50
samples 20000
RESAMPLE ▶
malloc p50
29 ns
malloc p99
121.6 µs
malloc max
9.84 ms
bump p99
12 ns
The histogram's x-axis is latency on a log scale, so each step right is ten times slower; the dashed lines mark the median (p50) and p99. Press RESAMPLE at the default fragmentation and the readouts land near the worked example's numbers: a median of about 29 ns, a p99 near 115 µs and a bump p99 of 12 ns. Drag fragmentation up and the right-hand hump grows. A simulator, not real measurements. The malloc model draws from a lognormal body (most samples near the median, a long tail to the right) (median around 30 ns) with a heavy-tailed mix that fires more often as the fragmentation slider rises, capturing free-list scans and lock waits. The bump model is a normal distribution clamped near 10 ns. Real numbers vary by allocator, platform, and heap state; the model only illustrates the shape, a tight body with a rare tail that runs orders of magnitude slower.
Four questions every allocator answers

The rest of the page walks one allocator per section, and each one is built to answer four questions in a different way. You can mostly predict an allocator's behavior from the answers:

    What sizes does it serve? One fixed size (pool, slab), a small set of size classes (TLSF, slab caches), powers of two (buddy), or anything (linear, free-list).
    What lifetime does it serve? A single frame (linear, stack), a single scope (stack with a marker), object-by-object (pool, slab, buddy, TLSF), or anywhere from a frame to the whole session (general-purpose).
    What's the worst-case time for one allocation? Constant (linear, stack, pool, slab fast path, TLSF), logarithmic in the heap size (buddy), or proportional to the length of the free list (first-fit, best-fit).
    What kind of waste does it produce? Alignment padding only (linear, stack), internal fragmentation only (pool, slab), internal plus external fragmentation (buddy), or mostly external (general-purpose free lists). Internal fragmentation is bytes wasted inside a block that's bigger than the request; external is free bytes split into holes too small to use.

Check Your Understanding Q1 of 2

A profiler shows your gameplay code calling new 14,000 times per frame and the p99 of those allocations is 850 microseconds. The p50 is fine. What is the most accurate diagnosis?
new is slow. Swap the system allocator for a faster general-purpose one like mimalloc or jemalloc, keeping every call site unchanged.
Per-frame allocations are paying for generality they never use. Replace them with a per-frame linear allocator that resets at the start of the next frame.
Schedule a full garbage-collection pass during loading screens, so gameplay frames inherit a freshly compacted heap.
0/2 correct

[Back to Memory Allocators from Scratch](index.md)
