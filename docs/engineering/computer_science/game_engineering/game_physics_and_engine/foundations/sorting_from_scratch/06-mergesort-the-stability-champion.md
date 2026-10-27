# Mergesort, the Stability Champion


Two sprites sit at exactly the same depth. Sorted with an unstable sort, they can swap places from one frame to the next, and the overlap flickers. A stable sort keeps equal keys in their original order, and mergesort is the classic stable sort. It's von Neumann's algorithm. Split the array in half; recursively sort each half; merge the two sorted halves into one. The merge step is the substance of the algorithm: two cursors walking forward, copying the smaller of the two current elements into the output. The recursion is log2n levels deep; each level does O(n) work; total n log n, worst case and average case both.

Mergesort's selling points over quicksort:

    Stable. Elements with equal keys end up in their original relative order. This is required for the "sort by department, then sort by name" pattern, and for draw-call sorts where ties are broken by submission order.
    Worst case is n log n. No adversarial input. Quicksort needs pivot tricks; mergesort needs nothing.
    External-memory friendly. Merge is purely sequential reads and writes, no random access. Sorts on tape and disk were almost always merge-based in the mainframe era, and external sorts of very large inputs still are.
    Parallelizes cleanly. Each half can be sorted on its own core with no synchronization until the merge.

The drawbacks are the reason it is not the default in-memory sort. Mergesort needs O(n) auxiliary memory for the merge buffer; quicksort sorts in place with O(log n) stack. The merge also moves every element at every level (into the buffer and back), while a partition swaps only the elements on the wrong side. For in-memory arrays of primitives a tuned quicksort usually beats a tuned mergesort, and std::sort doesn't have to be stable, so the C++ libraries use introsort there and keep mergesort for std::stable_sort.
top-down mergesort · C++

template<typename T>
void merge(T* values, T* buffer, int lo, int mid, int hi) {
  // Copy [lo, hi) to the scratch buffer; merge from the buffer back into values.
  for (int k = lo; k < hi; ++k) buffer[k] = values[k];
  int leftCursor = lo, rightCursor = mid;
  for (int writeAt = lo; writeAt < hi; ++writeAt) {
    if      (leftCursor  >= mid)                          values[writeAt] = buffer[rightCursor++];
    else if (rightCursor >= hi)                           values[writeAt] = buffer[leftCursor++];
    else if (buffer[rightCursor] < buffer[leftCursor])    values[writeAt] = buffer[rightCursor++];
    else                                                  values[writeAt] = buffer[leftCursor++];  // Ties take the left: stable.
  }
}

template<typename T>
void mergesort(T* values, T* buffer, int lo, int hi) {
  if (hi - lo < 2) return;              // Production: cut over to insertion sort below ~16.
  int mid = lo + (hi - lo) / 2;         // (lo + hi) / 2 can overflow int on huge arrays; this can't.
  mergesort(values, buffer, lo, mid);
  mergesort(values, buffer, mid, hi);
  merge(values, buffer, lo, mid, hi);
}

Worked example

Merge [2, 5, 9] with [3, 4, 10]. Compare the fronts: 2 < 3, take 2. 5 > 3, take 3. 5 > 4, take 4. 5 < 10, take 5. 9 < 10, take 9. The left side is empty, so copy 10.

Result [2, 3, 4, 5, 9, 10] after 5 comparisons for 6 elements. A merge of two runs never needs more comparisons than elements minus one.

The midpoint comment matters. In 2006 Joshua Bloch reported that the (lo + hi) / 2 in the Java standard library's binary search overflows once an array reaches 230 elements; the bug had sat in the JDK for nine years, and the binary search in Bentley's Programming Pearls (1986) had it too[17]. The fix, lo + (hi − lo) / 2, is the same one a mergesort midpoint needs.

Worked example

lo = 1,500,000,000 and hi = 1,600,000,000 both fit in a 32-bit int, whose maximum is 2,147,483,647. But lo + hi = 3,100,000,000 doesn't: it wraps to a negative number, and (lo + hi) / 2 is a negative index.

lo + (hi − lo) / 2 = 1,500,000,000 + 50,000,000 = 1,550,000,000, with no intermediate value above hi.
Live · The mergesort recursion tree
array size 16
Run ▶
Reset
levels
···
comparisons
···
aux memory
···
The full array is the top row and each row below splits it in half. Press Run: the leaves appear first, then each merge fills in its parent row, tinting each element by the half it came from. At the default 16 elements the readouts show 4 levels and 16 words of scratch memory. The tree has ⌈log2n⌉ levels; each level merges every element exactly once for O(n) work per level. Total: O(n log n). The aux-memory figure is n because the merge writes to a buffer the size of the active range. In-place merging exists (rotation-based, O(n log² n) overall), and std::stable_sort falls back to it when it can't allocate a buffer, but it is markedly slower. Timsort's merge needs a buffer only as large as the smaller of the two runs, a common compromise.

[Back to Sorting Algorithms](index.md)
