# Live Race: Every Sort, Side by Side


The sections above made claims about which sort wins on which data. This widget puts six of them on identical input so you can check. Six sorts run on the same input, animated as bar charts and paced by the work each one does. Pick the input shape to see how each algorithm responds to sorted, reverse-sorted, mostly sorted, and few-unique inputs, then press "Run race":
Live · Six sorts on identical input
input shape
array size 64
speed 1.0×
Run race ▶
Reset
insertion quicksort mergesort heapsort timsort radix
Every racer counts one step per comparison or per element write (a swap is one step, and so is each copy to or from a merge buffer), so the counters compare like with like. Radix finishes first on every shape here, at 4n steps (256 at the default n = 64): the keys are small integers below 256, so two 4-bit passes cover them and it never compares. On already-sorted input insertion sort and timsort tie at n − 1 comparisons. On reverse-sorted input timsort needs only about 1.5n steps (one run, reversed in place: 95 at n = 64) while insertion sort hits its worst case, n(n − 1)/2 comparisons plus as many shifts: 4,032 steps at n = 64. On random input quicksort is the fastest of the comparison sorts. The "mostly sorted" shape swaps n/8 random pairs, which scatters displaced elements across the whole array: at n = 64 insertion sort is still the fastest comparison sort, but by 128 the displacements add up and quicksort pulls ahead. This timsort is simplified (no minrun, no galloping), so it understates the real one on inputs like that.

[Back to Sorting Algorithms](index.md)
