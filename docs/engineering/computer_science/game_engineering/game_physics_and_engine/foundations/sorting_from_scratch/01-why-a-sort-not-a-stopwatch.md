# Why a Sort, Not a Stopwatch


A large scene can submit thousands to tens of thousands of draw calls per frame, and the renderer wants them grouped by render target, then shader, then material, then depth, so the GPU changes state as rarely as possible[1]. Sorting them with std::sort and a comparator that looks up each draw's material can eat over a millisecond of a 16.67 ms frame. Sorting the same draws by a packed integer key with a radix sort takes about a tenth of a millisecond.

Here's where the time goes. At 5,000 to 15,000 draws, a comparison sort does roughly 1.2 to 1.4 n log₂ n key comparisons, about 70,000 to 290,000. If the comparator dereferences a material or shader pointer, each comparison can cost tens of nanoseconds once the pointer chasing leaves the L1 cache, and the total runs past a millisecond. A four-pass radix sort over a packed 32-bit key does the same job in roughly 0.1 ms. Bevy measured this swap in 2022: on its many_cubes benchmark the median time of the opaque-3D sort phase dropped from 0.728 ms to 0.094 ms, though the same patch slowed two sprite benchmarks (§16)[2]. At this scale log₂ n is only about 12 to 14, so most of the gap comes from the key and the memory layout, not from the log factor.

Worked example

5,000 draws. log₂ 5,000 ≈ 12.3, so a comparison sort does about 1.2 × 5,000 × 12.3 ≈ 74,000 comparisons.

At 30 ns each (a pointer dereference that misses the cache, plus a short compare): 74,000 × 30 ns ≈ 2.2 ms.

A radix sort makes 4 passes over a packed 32-bit key at roughly 3 ns per element per pass: 4 × 3 ns × 5,000 = 0.06 ms, more than 30 times less.

This page walks through the family of sorts an engine programmer actually meets. Comparison sorts (quicksort, mergesort, heapsort, and their modern hybrids) are bounded below by Ω(n log n) for n distinct elements, a bound proved in §3. Non-comparison sorts (radix, counting, bucket) escape that bound by looking at the bits of the keys directly, which is why many renderers reach for them. Parallel sorts (bitonic, GPU radix) spread the work across thousands of GPU threads; bitonic sort even accepts more total work, O(n log² n), in exchange for a fixed, data-independent schedule.
The cost of the wrong sort

Whether that gap matters depends on how many items you sort and what one comparison costs. The widget below puts a comparison sort and a radix sort on the same axis: object count along the X, milliseconds per frame along the Y. The comparison curve is n log₂ n comparisons at an assumed ~30 ns each (a pointer dereference that misses cache plus a short string compare, ballparked from Norvig's latency numbers[3]). The radix curve is linear in n at ~3 ns per element per pass, four passes for a 32-bit key, plus roughly 2 µs of fixed setup (clearing and prefix-summing four 256-entry histograms). The crossover sits where that setup cost stops dominating: around n ≈ 100 with a 5 ns comparator, under twenty items with the 30 ns comparator (§12). With the 30 ns comparator, above 1,000 items the gap is an order of magnitude or more:
Live · Comparison vs radix at frame scale
both
comparison
radix
draw calls 5,000
per-comparison cost 30 ns
comparison sort
1.84 ms
radix sort (4 pass)
0.062 ms
frame budget at 60 fps
16.67 ms
The red curve is the comparison sort and the green one the radix sort, with the 60 fps frame budget dashed across the top; drag the draw-call count and the per-comparison cost, and read both costs below. At the defaults (5,000 draws, 30 ns) they read 1.84 ms and 0.062 ms. Both curves are cost models, not measurements. The comparison curve assumes every compare pays for a memory access; with keys packed into the array and data hot in L1, a compare costs a few nanoseconds (mostly branch mispredictions), so drag the slider toward its 5 ns minimum and the crossover moves right. The radix curve assumes a 32-bit packed key with one byte per pass; widening to 64 bits and 8 passes doubles its slope but does not change the shape.
Check Your Understanding Q1 of 2

Your renderer sorts 8,000 draw calls per frame using std::sort with a comparator that dereferences two material pointers and compares two strings. Profiling shows 1.2 ms in the sort. Which change reduces it the most?
Switch to a recursive mergesort instead of std::sort.
Pack the sort key into a single 64-bit integer (shader id, depth, etc.) and radix-sort on it.
Parallelize std::sort with eight threads.
0/2 correct

[Back to Sorting Algorithms](index.md)
