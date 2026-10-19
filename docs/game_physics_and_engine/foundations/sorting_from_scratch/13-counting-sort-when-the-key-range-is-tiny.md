# Counting Sort: When the Key Range Is Tiny


Particles come in 16 priority levels and you want them grouped by level every frame. With only 16 possible keys, you don't need to sort at all in the usual sense: count how many particles have each level, and you know exactly where each group starts. Counting sort is the inner pass radix sort wraps. Given an array of n keys, each in [0, k), sort in O(n + k): count occurrences of each value, prefix-sum the counts, scatter elements into output. When k = O(n) the cost is linear; when k is much larger than n the histogram-allocation cost dominates and counting sort loses to a comparison sort.

Worked example

Keys [2, 0, 3, 2, 1, 0], each in [0, 4).

Count: key 0 appears 2 times, key 1 once, key 2 twice, key 3 once: [2, 1, 2, 1]. Prefix-sum into starting offsets: [0, 2, 3, 5].

Scatter in input order: the first 2 goes to slot 3, the 0 to slot 0, the 3 to slot 5, the second 2 to slot 4, the 1 to slot 2, the last 0 to slot 1. Result [0, 0, 1, 2, 2, 3], stable, in two passes over the data.

Use cases in engines:

    Particle priority bucketing. A particle system with 4-bit priority gives 16 buckets; one counting pass groups particles per frame in O(n).
    Material ID grouping. If material IDs are dense in [0, 256), counting sort by material ID puts every draw call of the same material together with one pass.
    Histogram-based effects. Auto-exposure's luminance histogram is the counting half of a counting sort on log-luminance; its prefix sums give the percentiles the exposure logic needs.

Java's Arrays.sort switches to counting sort for byte arrays longer than 64 elements and char or short arrays longer than 1,750, because the key ranges (256 and 65,536 values) make the histogram cheap once it is amortized over enough elements[35]. For int and wider types it uses dual-pivot quicksort, since a histogram over 4 billion values is out of the question.

[Back to Sorting Algorithms](index.md)
