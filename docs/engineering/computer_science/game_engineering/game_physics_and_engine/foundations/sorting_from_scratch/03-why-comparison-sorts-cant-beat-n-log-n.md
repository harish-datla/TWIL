# Why Comparison Sorts Can't Beat n log n


You've tuned a comparison sort's pivot and cutoffs, and it still costs too much. Is there a cleverer comparison sort out there? Not one that needs asymptotically fewer comparisons, and knowing that tells you when to stop tuning and change the approach. Any algorithm that sorts by comparing pairs of elements (no peeking at their bits, no precomputed table) must do about n log2 n comparisons in the worst case. The proof is a short counting argument, and it also explains how radix sort and counting sort get under the bound: they break its one assumption.

Model a comparison sort as a decision tree: the root is the first comparison the algorithm makes; the two children are the two possible outcomes; each leaf is one possible final permutation. For an input of n distinct elements there are n! possible permutations, so the tree needs at least n! leaves. A binary tree with L leaves has height at least log2 L. So the height (the worst-case number of comparisons on some input) is at least log2(n!).

Worked example

n = 3: there are 3! = 6 orderings. Each comparison has two outcomes, so 2 comparisons can tell apart at most 4 orderings and 3 can tell apart 8. At least 3 comparisons are needed.

n = 10: 10! = 3,628,800 orderings, and log2 3,628,800 ≈ 21.8, so at least 22.

n = 5,000: log2(5,000!) ≈ 54,200 comparisons at the very least. The 74,000 in §1 is within 40% of that floor, so no comparison sort will do this job several times faster.

The equation turns "log2(n!)" into something you can compare with n log n. It uses a Stirling-type bound, n! > (n/e)n for n ≥ 1[18]:
eq. 1 · the lower bound T(n) ≥ log2(n!) ≥ log2((n/e)n) = n log2(n) − n log2(e) = Ω(n log n)

In words: every comparison sort needs at least about n log2 n − 1.443 n comparisons on some input. The 1.443 n part grows more slowly than n log n, so for large n the floor is essentially n log n. Mergesort and heapsort match this bound up to a constant factor in the worst case; quicksort matches it on average.

The widget below builds the decision tree for n = 3 by hand. There are 3! = 6 permutations, so the tree has 6 leaves and minimum height ⌈log2(6)⌉ = 3 comparisons. Click a permutation to highlight the path:
Live · The decision tree for n = 3
trace permutation
leaves needed
3! = 6
min height
⌈log₂6⌉ = 3
path to this leaf
2
Each inner node is one comparison and each leaf one of the six orderings of a, b and c. Pick an ordering in the dropdown to highlight the comparisons that lead to it, and read the path length below. Any comparison sort on three elements needs a tree with at least six reachable leaves, so some input costs it three comparisons; this tree reaches two of its leaves in two. The worst case is the tree's height. For n = 3 the optimum is 3 comparisons; for n = 10 the bound is ⌈log2(10!)⌉ = 22 and 22 is achievable. Ford and Johnson's 1959 merge insertion sort matches ⌈log2 n!⌉ for every n up to 11[19]; it is rarely used because its bookkeeping costs more than the comparisons it saves.


[Back to Sorting Algorithms](index.md)
