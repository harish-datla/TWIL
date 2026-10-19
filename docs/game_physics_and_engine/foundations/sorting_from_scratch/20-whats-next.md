# What's Next


Next in the series, x86-64 Assembly goes below the C++ to the instructions the CPU runs: registers, calling conventions, branches and the out-of-order core, which is where branchless partitions like pdqsort's earn their speed. Beyond sorting itself, the natural follow-ups, in roughly increasing depth:

    Selection algorithms. "Find the k-th smallest" without sorting the whole array. Quickselect partitions like quicksort but recurses into only one side, for expected O(n). Median-of-medians (BFPRT, 1973) gives a deterministic linear-time selection. The introselect fallback in Musser 1997 is to selection what introsort is to sorting.
    External-memory sorting. Data larger than RAM. k-way merge sort plus a tape-or-disk layout. The classic algorithm; on NVMe SSDs bandwidth matters more than seek time, which changes the tuning.
    Parallel and concurrent sorts. Multi-way mergesort with work stealing, parallel quicksort, GPU radix sort. The work-span model (as in Cilk) for analyzing them. The job systems tutorial introduces the runtime; the parallel-sort algorithms layer on top.
    Top-k and partial sort. When you don't need the whole order, just the smallest k. std::partial_sort is typically heap-based (O(n log k)); when k is a large fraction of n, std::nth_element followed by sorting the first k is usually faster.
    Sorting under adversarial input. McIlroy's adversary (§5) shows how an attacker who controls the input can drive a deterministic quicksort quadratic. Introsort's depth limit or pdqsort's heapsort fallback caps the damage at O(n log n); randomized pivots make the attack depend on guessing the random state.
    Cache-oblivious sorting. Sorting N items with a cache of size M and blocks of size B needs Θ((N/B) · logM/B(N/B)) block transfers; Frigo et al.'s funnelsort meets that bound without knowing M or B[31]. It is the model to use when the data is far larger than cache.

The most useful next step is empirical: profile your renderer's draw-call sort. If its comparator dereferences pointers or compares strings, pack an integer key, then measure a radix sort against std::sort on the packed keys. 

[Back to Sorting Algorithms](index.md)
