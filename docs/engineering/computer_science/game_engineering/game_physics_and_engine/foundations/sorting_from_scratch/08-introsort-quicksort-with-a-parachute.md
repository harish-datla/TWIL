# Introsort: Quicksort with a Parachute


A library sort can't know what input it'll get, and some inputs are built on purpose to defeat quicksort's pivot choice. David Musser's 1997 introsort[9] lets a library keep quicksort's speed and still promise O(n log n) in the worst case, which C++11 made a requirement for std::sort. Run quicksort, count recursion depth, and if it exceeds 2 log2n, finish the current subarray with heapsort. Quicksort's average case stays intact; heapsort's worst-case bound caps the whole algorithm. Add an insertion-sort cutoff at small n and you have the std::sort in libstdc++ (since the GCC 3.x era) and in MSVC's STL. libc++ added the depth limit in LLVM 14 (2022)[22]; LLVM 17 (2023) moved it to a pdqsort-derived implementation with branchless partitioning for arithmetic types (§10)[10].
introsort · C++

#include <algorithm>  // std::iter_swap, std::make_heap, std::sort_heap
#include <bit>        // std::bit_width (C++20)

// Returns whichever of the three iterators points at the median value.
template<typename It, typename Compare>
It median_of_three(It lowSample, It midSample, It highSample, Compare less) {
  if (less(*lowSample, *midSample)) {
    if (less(*midSample, *highSample)) return midSample;            // low < mid < high
    return less(*lowSample, *highSample) ? highSample : lowSample;  // mid is the largest
  }
  if (less(*lowSample, *highSample)) return lowSample;              // mid <= low < high
  return less(*midSample, *highSample) ? highSample : midSample;    // low is the largest
}

template<typename It, typename Compare>
void introsort_loop(It first, It last, int depthBudget, Compare less) {
  while (last - first > 16) {            // Ranges of 16 or fewer wait for the final insertion sort.
    if (depthBudget == 0) {              // Quicksort recursion is going too deep:
      std::make_heap(first, last, less); // finish this range with heapsort (§7),
      std::sort_heap(first, last, less); // which is O(n log n) on any input.
      return;
    }
    --depthBudget;
    It middle = first + (last - first) / 2;
    std::iter_swap(first, median_of_three(first, middle, last - 1, less));  // Pivot to the front.
    It cut = hoare_partition(first, last, less);
    introsort_loop(cut, last, depthBudget, less);  // Recurse on the right part...
    last = cut;                                    // ...and loop on the left, saving a call.
  }
}

template<typename It, typename Compare>
void introsort(It first, It last, Compare less) {
  const auto count = static_cast<std::size_t>(last - first);
  // Musser's budget: 2 * floor(log2 n) levels of partitioning before heapsort takes over.
  int depthBudget = 2 * (static_cast<int>(std::bit_width(count)) - 1);
  introsort_loop(first, last, depthBudget, less);
  insertion_sort(first, last, less);     // Every element is now within 16 slots of home: one cheap pass.
}

Worked example

Sorting 1,000,000 elements: std::bit_width(1,000,000) is 20, so the budget in the listing is 2 × (20 − 1) = 38 levels of partitioning.

A perfectly balanced quicksort needs about 20 levels, so ordinary inputs finish well inside the budget. An input that keeps forcing lopsided splits uses up the 38 levels on some subarray, and that subarray, and only that one, gets finished by heapsort.

Musser's killer-sequence experiment is worth repeating: he constructed a 100,000-element input designed to defeat median-of-three quicksort and measured introsort's running time at 1/200 of the median-of-three quicksort's[9]. Heapsort takes over only for the subarrays that exhaust the depth budget and finishes them in n log n; the rest of the input keeps quicksort's average-case behavior.
Why 2 log2n?

The constant in front of the log is a tradeoff. A perfectly balanced quicksort recurses exactly log2n deep; a median-of-three quicksort on random data goes somewhat deeper on some branches. The factor 2 gives quicksort room to finish on inputs that are unlucky but not pathological. At 1.0, heapsort would trigger on ordinary inputs; at 4.0, an adversarial input would burn roughly twice as much partitioning work before heapsort took over (the bound stays O(n log n) either way, only the constant grows). Musser suggested 2 ⌊log2n⌋; libstdc++ and libc++ use it, and MSVC allows about 1.5 log2n.

Many equal keys are a weak spot. Hoare partitioning (both cursors stop on elements equal to the pivot) splits a run of equal keys evenly, so it stays n log n, but it still spends n log n work on input that a three-way partition would finish in linear time; a Lomuto partition is worse and goes quadratic. Bentley and McIlroy 1993 popularized the three-way ("fat") partition for this case, and pdqsort gets the same effect with a cheaper trick (§10), one reason it beats plain introsort on inputs with many duplicates.

[Back to Sorting Algorithms](index.md)
