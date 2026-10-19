# Bitonic Sort and the GPU


A GPU particle system has its particles in GPU memory already, and copying them back to the CPU to sort would cost more than the sort. The sort has to run on the GPU. Quicksort and a sequential mergesort make one decision at a time, and a GPU wants thousands of threads doing identical work in lockstep. Two families fit: radix sorts built on parallel prefix sums (each pass's histogram and scatter split cleanly across threads; GPU sorts of large key arrays, such as the one in CUDA's CUB library, are usually radix sorts), and sorting networks, where which position compares with which is fixed ahead of time. Ken Batcher's 1968 bitonic sort[27] is the classic sorting network.

The structure: a bitonic sequence is one that increases then decreases (or a rotation of one). One layer of n/2 parallel compare-and-swaps, pairing element i with element i + n/2, splits a bitonic sequence into two bitonic halves with every element of the first no larger than any element of the second; repeating on the halves sorts it in log2 n layers (the "bitonic merge"). Bitonic sort builds bitonic sequences by sorting halves in opposite directions and merging, recursively. Total operations: O(n log²n). Depth (the longest chain of dependent layers): O(log²n). With enough threads, depth matters more to wall time than total work.

Bitonic sort's cost summary versus the CPU sorts:
Sort	Total work	Parallel depth	In place	Stable
quicksort (serial)	O(n log n)	O(n log n)	yes	no
mergesort (serial)	O(n log n)	O(n log n)	no	yes
radix sort (serial)	O(d·n)	O(d·n)	aux	yes
bitonic sort (parallel)	O(n log²n)	O(log²n)	yes	no
GPU radix sort	O(d·n)	O(d log n)	aux	yes

Bitonic sort does more total work than mergesort (n log²n vs n log n) but every compare-and-swap in a layer is independent. Sorting 4096 elements takes 78 layers of 2,048 compare-and-swaps each; a serial mergesort needs about 50,000 dependent comparisons. The extra work buys a schedule with no data-dependent control flow. GPU Gems 2, chapter 46, describes a GPU implementation[28]. For large arrays radix sort usually wins on the GPU as well: Unreal depth-sorts its GPU particles with a compute-shader radix sort (GPUSort.cpp and RadixSortShaders.usf in the engine source).
Live · The bitonic sort network for n = 8
array size (power of 2)
Run ▶
Reset
total ops
···
parallel depth
···
speedup, n/2 threads
···
Each vertical wire is one array position and each layer of arrows one round of compare-and-swaps. Press Run to fire the network a layer at a time; at n = 8 the readouts show 24 operations in 6 layers, a 4.0× speedup with 4 threads. Each layer of the network can run in parallel: every compare-and-swap in a layer touches a disjoint pair of wires. The depth sets the wall time on a parallel machine; the total op count sets the energy spent. With n/2 threads, one per compare-and-swap in a layer, the speedup over running the same network serially is total ops ÷ depth. For n = 4096 the network has 78 layers and about 160,000 compare-and-swaps in total.

[Back to Sorting Algorithms](index.md)
