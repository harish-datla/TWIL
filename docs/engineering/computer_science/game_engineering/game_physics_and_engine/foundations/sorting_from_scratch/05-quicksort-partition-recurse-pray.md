# Quicksort: Partition, Recurse, Pray


A sort that handles a million random keys in milliseconds can take far longer on the same keys already in order, if it picks its pivots badly. Quicksort is the fast algorithm with that hazard built in. Tony Hoare's quicksort, from his 1959 stint at Moscow State University, is the basis of most general-purpose unstable sorts in standard libraries (introsort, pdqsort, dual-pivot quicksort). Pick a pivot, partition the array into "less than pivot" and "greater than pivot," and recurse on each side. The expected cost is n log n and the worst case is n²; which one you get depends on how the pivot is chosen.
The partition

Two partition schemes ship in production code. Hoare's partition does fewer swaps; Lomuto's partition is simpler to write correctly. The C++ standard libraries use tuned variants of Hoare's; branchless Lomuto variants have come back in newer sorts such as Rust's current sort_unstable. A Hoare partition in C++:
Hoare partition · C++

// Partitions [first, last) around the value of *first and returns a cut:
// every element of [first, cut) is <= pivot, every element of [cut, last)
// is >= pivot, and neither side is empty. Requires last - first >= 2.
template<typename It, typename Compare>
It hoare_partition(It first, It last, Compare less) {
  auto pivot = *first;           // Copy it: the swaps below move the original. Production code
                                 // first swaps a median-of-3 (or ninther) into *first.
  It leftCursor  = first;
  It rightCursor = last - 1;
  while (true) {
    while (less(*leftCursor, pivot)) ++leftCursor;    // Stop on something >= pivot.
    while (less(pivot, *rightCursor)) --rightCursor;  // Stop on something <= pivot.
    if (leftCursor >= rightCursor) return rightCursor + 1;  // Cursors met or crossed; done.
    std::iter_swap(leftCursor, rightCursor);          // Swap the out-of-place pair...
    ++leftCursor;                                     // ...and step past it. The swapped
    --rightCursor;                                    // values stop the next scans in bounds.
  }
}

The recursion is the easy part; quicksort variants differ mostly in how they pick the pivot.
The pivot problem

Worked example

Sort 48 already-sorted keys with the first element as pivot (the widget below). Each pivot is the smallest key left, so each partition peels off one element: 47 + 46 + … + 1 = 1,128 comparisons and a recursion 47 deep.

Median-of-three on the same input picks the true middle every time: 183 comparisons and depth 5.

If the pivot is the median of the data, quicksort halves the array each call and runs in n log n. If the pivot is the minimum (or maximum) every time, one side is empty, the recursion depth approaches n, and the cost is quadratic; for large n the call stack can overflow too, unless the implementation recurses on the smaller side and loops on the larger.

Pivot strategies in roughly increasing sophistication:

    First element. Trivial. Worst case on already-sorted input, a common shape in real data. This is the textbook version; production libraries avoid it.
    Random element. Expected n log n on every input, and no fixed input can force quadratic behavior. Libraries mostly prefer median-of-three anyway: it splits more evenly on average and keeps the sort deterministic.
    Median-of-three. Sample the first, middle, and last element; use their median. Defeats sorted and reverse-sorted inputs. Bentley and McIlroy's 1993 sort uses it for mid-sized arrays (8 to 40 elements in their tuning)[8].
    Pseudomedian-of-nine ("ninther"). For large arrays, sample three groups of three, take the median of each, then the median of those medians. Bentley and McIlroy again, borrowing Tukey's ninther. Much harder to hit by accident.
    Median of medians (BFPRT). Compute the exact median in O(n). Guarantees n log n worst case, but the constant is large enough that it is rarely used for sorting.

Modern hybrids (introsort, pdqsort) give up on pivot perfection and accept that any deterministic strategy can be defeated. They monitor recursion depth or partition quality and take a fallback path when things go wrong: introsort switches to heapsort (§8); pdqsort first shuffles a few elements to break up patterns and falls back to heapsort if bad partitions keep coming (§10).

The widget below shows quicksort running on data you control. Switch the pivot strategy and the input shape to see when quicksort runs at n log n and when it falls apart:
Live · Quicksort with different pivots
pivot strategy
input shape
size 48
Run ▶
comparisons
···
max recursion depth
···
vs n log n
···
The bars are the array; the highlighted bar is the current pivot, swaps flash orange and finished pivots turn green. Pick a pivot strategy, an input shape and a size, then press Run. The widget runs a Lomuto-partition quicksort and counts comparisons against the pivot. Try "first element" with "already sorted": the comparisons climb to n(n−1)/2 and the recursion depth to n − 1, the textbook quadratic worst case. Switch to "median-of-three" and the same sorted input splits evenly every time. Then try "organ-pipe" (1, 3, 5, …, max, …, 4, 2) with median-of-three: at n = 96 the depth reaches 28 instead of 6 and the comparison count about a quarter of n(n−1)/2, growing quadratically. That is a bad case for this simple implementation rather than a universal killer; McIlroy showed in 1999 that for essentially any deterministic pivot rule an adversary can build a fully quadratic input[20], which is why production sorts add a fallback.

[Back to Sorting Algorithms](index.md)
