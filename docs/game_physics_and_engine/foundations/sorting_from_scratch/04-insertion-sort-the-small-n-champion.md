# Insertion Sort, the Small-n Champion


A quicksort that recurses all the way down to one-element ranges spends most of its calls on tiny subarrays, where picking a pivot and recursing cost more than the sorting. Every library sort stops recursing at some small size and finishes with insertion sort instead. Insertion sort is what most production sorts fall back to at small sizes. Its worst case is O(n²), which is easy to file under "bad" and move on, but that misses the small-n regime: insertion sort usually wins below roughly 16 to 32 elements on current CPUs, which is exactly the size of the subarrays a recursive sort produces at the bottom of its recursion. The reasons are mechanical, not asymptotic: a tight inner loop, no recursion or pivot selection, and sequential memory access that stays in L1.
insertion sort · C++

template<typename It, typename Compare>
void insertion_sort(It first, It last, Compare less) {
  if (first == last) return;                      // first + 1 would step past an empty range.
  // Outer loop: bring *cursor into its place among the sorted prefix [first, cursor).
  for (It cursor = first + 1; cursor != last; ++cursor) {
    auto key = std::move(*cursor);                // Save the element we're placing.
    It target = cursor;
    // Walk left while the element on the left is bigger than key.
    while (target != first && less(key, *(target - 1))) {
      *target = std::move(*(target - 1));        // Shift the bigger one right by one slot.
      --target;
    }
    *target = std::move(key);                     // Drop key into the gap.
  }
}

Cost analysis: the outer loop runs n − 1 times; the inner loop walks back as far as the new element needs to go. On a fully sorted input the inner loop exits immediately and the total cost is O(n). On a reverse-sorted input the inner loop walks all the way back every time and the total cost is n(n−1)/2 = Θ(n²). The average case on random data is also Θ(n²), about n²/4 shifts: on average half of the elements to the left of each new element are larger than it, so the average is half the worst case.

Worked example

16 elements. Already sorted: each of the 15 new elements is compared once and stays put, so 15 comparisons and no shifts.

Reverse sorted: the i-th new element shifts past all i before it, so 1 + 2 + … + 15 = 120 shifts.

Random: about half of that, 16² / 4 = 64 shifts on average. At this size those 64 shifts move data that's already in L1.

The widget below races insertion sort against a simple recursive quicksort (median-of-three pivot, Lomuto partition) at array sizes from 4 to 256. The crossover is what production cutoffs are tuned to:
Live · Where insertion sort beats quicksort
input shape
Run race ▶
crossover n
···
insertion at n=16
···
quick at n=16
···
Each array size gets two bars, insertion sort and quicksort, measuring the same input. Press Run race, then switch the input shape and run again; the readouts give the crossover size and both costs at n = 16 (126 against 192 on random input). Cost is counted in array reads and writes, not time, and the quicksort's function-call overhead isn't charged, so real crossovers sit somewhat higher than the one shown (24 on random input). Switch input shape to see the crossover move: insertion sort never loses on already-sorted input (it runs in linear time), and on "few unique" it stays ahead longer because this Lomuto quicksort handles duplicate keys badly. Cutoff sizes used in production: libstdc++ below 16, current libc++ below 24 (30 for trivially copyable types in LLVM 14 to 16)[10], MSVC 32, Java's DualPivotQuicksort 44 (47 through JDK 13), pdqsort 24, Go 12.


[Back to Sorting Algorithms](index.md)
