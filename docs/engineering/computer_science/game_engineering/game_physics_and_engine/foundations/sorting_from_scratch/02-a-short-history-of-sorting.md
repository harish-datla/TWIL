# A Short History of Sorting


Machine sorting predates electronic computers by half a century. Herman Hollerith's punched-card equipment tabulated the 1890 US Census, and the electromechanical card sorters that followed it ordered a deck one column at a time, least significant digit first: LSD radix sort[4]. You don't need the rest of the history to pick a sort, but two patterns from it explain what the library sorts look like today.

First, the major standard-library sorts are hybrids: quicksort plus insertion sort plus a heapsort fallback (introsort, pdqsort), or mergesort plus insertion sort on short runs plus galloping merges (timsort, powersort). Second, the famous bugs were in the bookkeeping around the comparison loop, not the loop itself: the 2015 timsort stack-invariant bug was in the merge-stack logic, and the binary-search overflow Joshua Bloch reported in 2006[17] was in the midpoint arithmetic.
Timeline: von Neumann's merge sort to Rust's ipnsort

[Back to Sorting Algorithms](index.md)
