# Picking the Right Sort


When you're about to call a sort in engine code, start from what you know about the data: how many items, what the key is, whether order already exists, where the data lives. The decision table:
Workload	Right answer	Why
Generic C++, types with cheap compare	std::sort (introsort / pdqsort)	Fast on average, n log n worst case, in place, library-provided.
Generic C++, types with expensive compare	a merge-based sort (std::stable_sort, timsort)	Close to the minimum number of comparisons, about n log₂ n, versus roughly 1.2 to 1.4 n log₂ n for quicksort variants.
Generic, must be stable	std::stable_sort (a mergesort in libstdc++, libc++, and MSVC)	The unstable sorts don't preserve equal-key order.
Real data, often partially sorted	timsort (Python, Java, V8) or powersort	Adaptive: O(n) on already-sorted input, and close to linear when the data is a few long runs.
Bounded integer keys, n > 100	radix sort (LSD)	Linear time, no comparisons, parallelizes well. Common for renderer sort keys.
Small integer range (k no larger than about n)	counting sort	One counting pass and one scatter, no comparisons. Java does this for large byte, char, and short arrays.
n < 32, any keys	insertion sort	Recursive sorts pay more overhead than the inner-loop work at this scale.
Worst-case latency budget (real-time)	heapsort or introsort	Guaranteed n log n; quicksort alone can blow the budget on bad input.
Data on GPU	bitonic sort or GPU radix	Both fit GPU parallelism. Radix usually wins for large arrays of fixed-width keys; bitonic suits small batches and arbitrary comparisons.
Data on disk, larger than RAM	external mergesort	Sequential I/O; a k-way merge needs only logk of the run count in passes over the data.
Strings of varied length	MSD radix or three-way radix quicksort	Looks at each character of the distinguishing prefixes a small number of times instead of re-comparing shared prefixes on every comparison.

std::sort is the right default for most sorts. The exceptions are large sorts that run every frame on keys that pack into integers (draw calls, particles) and sorts of nearly sorted data (the broad phase), which is where the case studies above come from.

[Back to Sorting Algorithms](index.md)
