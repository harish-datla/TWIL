# pdqsort, the Branchless Modern Quicksort


On random keys, quicksort's partition loop asks "is this element smaller than the pivot?" and the answer is a coin flip, so the CPU's branch predictor guesses wrong about half the time and throws away work each time. On cheap integer keys that misprediction costs more than the comparison itself. Pattern-defeating quicksort, Orson Peters (the 2021 paper[16] writes up an algorithm that had been shipping since 2017), is a descendant of introsort. It adds three ideas that make it faster than plain introsort on many common inputs[16]:

    BlockQuicksort partitioning. Edelkamp and Weiß 2016[21] showed that on cheap keys the partition's unpredictable branch on the pivot comparison costs more than the comparison itself. Their fix: scan a block of elements (128 in BlockQuicksort, 64 in pdqsort) and record the offsets of the ones on the wrong side in a small buffer, using arithmetic instead of branches; then swap the recorded pairs in a separate pass. libc++'s version records the same information as 64-bit bitsets. Data-dependent mispredictions drop to nearly zero, leaving only predictable loop branches. BlockQuicksort measured an 80% speedup over GCC's std::sort on random integers.
    Pattern defeat. Pdqsort checks whether each partition was badly unbalanced (either side under n/8). If so, it deterministically swaps a few elements at fixed offsets to break up patterns such as organ pipes or sawtooths before continuing. That handles accidental bad inputs, not a determined adversary; for those, after about log2 n bad partitions it falls back to heapsort the way introsort does.
    Equal keys. In every subarray except the leftmost, each element is at least as large as the element just before the subarray, an earlier pivot. If the new pivot equals that earlier pivot, pdqsort partitions into "equal to pivot" and "greater" instead and skips the equal part entirely. With k distinct values the sort runs in O(n · k), linear when k is small.

Pdqsort was Rust's slice::sort_unstable from 2017 until Rust 1.81 (2024) replaced it with ipnsort, a pdqsort descendant by the same author and Lukas Bergdoll[32]; it ships in Boost.Sort, and it is the basis of the branchless std::sort that Google's engineers contributed to libc++ (described in 2022, shipped in LLVM 17, 2023). The libc++ 17 release notes report std::sort getting up to 50% faster for arithmetic types and about 10% faster for other types[22].

The branchless scan is short. The conventional Hoare partition has a data-dependent branch on the pivot comparison, and on random data the predictor is wrong about half the time. BlockQuicksort instead fills two arrays of offsets (the positions of wrong-side elements found in a left block and a right block), then swaps them in pairs. The scan loop has no data-dependent branch because the comparison result is added to a counter instead of being branched on:
block partition (sketch) · C++

template<typename T>
void block_scan_left(T* block, T pivot,
                     unsigned char* offsets, int& count) {
  count = 0;
  for (int i = 0; i < 64; ++i) {
    offsets[count] = (unsigned char)i;
    count += (block[i] >= pivot);   // No branch: bool-to-int extends the count.
  }
}
// Symmetric block_scan_right. Then a second loop swaps element pairs from the
// two offset arrays; that pass has one predictable loop branch and no data-dependent ones.

Worked example

Run the scan above on an 8-element block [7, 2, 9, 5, 1, 6, 3, 8] with pivot 5 (shortened from 64). Every iteration writes its index into offsets[count], then adds 1 to count if the element is ≥ 5.

The elements ≥ 5 are at indices 0, 2, 3, 5 and 7, so count ends at 5 and offsets starts 0, 2, 3, 5, 7. The writes for 2, 1 and 3 were simply overwritten by the next index. No iteration branched on the data.

Combined effect: pdqsort keeps introsort's O(n log n) worst case (both fall back to heapsort), is substantially faster on random integers thanks to the branchless partition, is much faster on inputs with few distinct keys, and runs in linear time on fully ascending, fully descending, and all-equal input, which introsort handles in n log n[16]. Nearly sorted input also benefits: when a partition needed no swaps, pdqsort tries a bounded insertion sort on each side and keeps the result if it finishes quickly.

[Back to Sorting Algorithms](index.md)
