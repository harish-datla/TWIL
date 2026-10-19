# Heapsort, the Worst-Case Insurance Policy


Some code can neither allocate a scratch buffer nor risk a quadratic blow-up: a kernel, an embedded C library, or quicksort's own emergency exit. It needs a sort that's in place and guaranteed n log n. Heapsort, J.W.J. Williams 1964[6], is the in-place worst-case-n log n sort. Quicksort can degrade to quadratic; mergesort needs n auxiliary words; heapsort needs neither. Production code mostly uses heapsort as a fallback: introsort calls it when recursion depth exceeds its budget. Code that can neither allocate a merge buffer nor risk quadratic time, such as some kernel and embedded C libraries, uses it directly.

The algorithm has two phases. First, build a max-heap in place: the largest element ends up at index 0. Then repeatedly swap the root (largest remaining) to the end of the array and sift down what landed at the root to restore the heap property. Each swap-pop step takes O(log n) sift-down time, done n times: O(n log n) total.

The non-obvious part is the build. The naive method (insert n elements one by one) is n log n. Floyd's bottom-up heapify, published the same year as Williams' paper[7], is O(n). The proof is a geometric sum: half the array is at the leaves and needs zero sift-down, a quarter is at level h−1 and needs at most one sift-down step, an eighth needs at most two, and the sum Σ k · 2−k is bounded by a constant.

Worked example

A full heap of 1,023 elements has 10 levels. The 512 leaves need no sift-down. The 256 nodes above them sink at most 1 level, the 128 above those at most 2, and so on up to the root, which sinks at most 9.

Total: 256 × 1 + 128 × 2 + 64 × 3 + 32 × 4 + 16 × 5 + 8 × 6 + 4 × 7 + 2 × 8 + 1 × 9 = 1,013 sift-down steps at most, fewer than one per element. Inserting the elements one by one could cost up to 9 steps each, about 9,000.
heapsort with Floyd's heapify · C++

template<typename T>
void sift_down(T* values, int root, int heapEnd) {
  // Walk root down toward the leaves; at each step, swap with the larger child if needed.
  while (true) {
    int leftChild  = 2 * root + 1;
    int rightChild = 2 * root + 2;
    int biggest    = root;
    if (leftChild  < heapEnd && values[leftChild]  > values[biggest]) biggest = leftChild;
    if (rightChild < heapEnd && values[rightChild] > values[biggest]) biggest = rightChild;
    if (biggest == root) return;                // Heap property restored.
    std::swap(values[root], values[biggest]);
    root = biggest;                             // Follow the swap down.
  }
}

template<typename T>
void heapsort(T* values, int count) {
  // Phase 1: build a max-heap in place, bottom-up. Floyd's O(n) heapify.
  for (int i = count / 2 - 1; i >= 0; --i) sift_down(values, i, count);
  // Phase 2: extract the max count - 1 times, growing a sorted suffix.
  for (int heapEnd = count - 1; heapEnd > 0; --heapEnd) {
    std::swap(values[0], values[heapEnd]);      // Largest remaining moves to its final slot.
    sift_down(values, 0, heapEnd);              // Restore the heap on the shrinking prefix.
  }
}

The catch is speed. Heapsort is n log n in the worst case, but Musser reports it taking 2 to 5 times as long as quicksort on most inputs[9]. Two common causes: sift-down makes about two comparisons per level with unpredictable outcomes, and its parent-to-child jumps double in stride at every level, so once the heap outgrows a cache level those reads pay the next level's latency. Quicksort's partition streams through contiguous memory that the prefetcher handles well. Heapsort earns its place where the worst-case guarantee matters more than average speed.
Live · Heapsort step-by-step
size 16
speed 1.0×
Run ▶
Reset
phase
···
comparisons
···
swaps
···
The top half is the array and the bottom half the same array drawn as a tree, index 0 at the root. Press Run and watch the build phase sift nodes down, then the extract phase move the root to the growing sorted suffix on the right; the readouts count comparisons and swaps. Phase 1 ("build") starts from index ⌊n/2⌋−1 (the last internal node) and works backward toward the root, sifting each one down. The leaves (half the array) never start a sift-down, and most internal nodes sit near the bottom and sink only a level or two: that is why Floyd's build is O(n). Phase 2 ("extract") swaps the root to the end and re-sifts; each step is one O(log n) sift-down, and this phase does most of the comparisons.

[Back to Sorting Algorithms](index.md)
