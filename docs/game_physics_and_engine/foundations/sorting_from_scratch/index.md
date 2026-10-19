# Sorting Algorithms

Every frame your engine sorts something: draw calls by shader and depth, transparent sprites back to front, particles, physics bounding boxes. Pick the wrong sort, or the wrong key, and a job that should take a tenth of a millisecond takes several. This page shows you how to choose: what the library sorts actually do (introsort, pdqsort, timsort and their parts), when a radix sort on a packed integer key beats them all, and how to build that key. The core routines are written from scratch in C++, every one runs live in the browser, and the page ends with case studies from shipping engines.
Time
~70 min
Level
Junior to mid; review for senior
Prereqs
You can read C++ or pseudocode at intermediate level. Big O notation (read the prior tutorial if it feels rusty).
Hardware
None, but a feel for cache and branch prediction helps in §10 and §16.
◂ Build a Game Engine
Phase 0 · Foundations
Next · x86-64 Assembly ▸
What you'll be able to do by the end

    Estimate what a sort costs per frame from its comparison count and the cost of one comparison, and spot when the comparator, not the algorithm, is the problem.
    Pick the right sort for a job: std::sort, a stable sort, timsort for nearly sorted data, radix or counting sort for integer keys, a GPU sort for GPU data.
    Pack draw calls into a 64-bit sort key that orders opaque draws front to back and transparent ones back to front, including float depth.
    Explain how introsort and pdqsort guard against quicksort's quadratic worst case, and why every library sort switches to insertion sort for small ranges.
    Avoid the comparator bugs that crash or corrupt std::sort: NaN keys, inconsistent ordering, overflow in a - b.


## Sections

- [Why a Sort, Not a Stopwatch](01-why-a-sort-not-a-stopwatch.md)
- [A Short History of Sorting](02-a-short-history-of-sorting.md)
- [Why Comparison Sorts Can't Beat n log n](03-why-comparison-sorts-cant-beat-n-log-n.md)
- [Insertion Sort, the Small-n Champion](04-insertion-sort-the-small-n-champion.md)
- [Quicksort: Partition, Recurse, Pray](05-quicksort-partition-recurse-pray.md)
- [Mergesort, the Stability Champion](06-mergesort-the-stability-champion.md)
- [Heapsort, the Worst-Case Insurance Policy](07-heapsort-the-worst-case-insurance-policy.md)
- [Introsort: Quicksort with a Parachute](08-introsort-quicksort-with-a-parachute.md)
- [Timsort, the Natural-Runs Mergesort](09-timsort-the-natural-runs-mergesort.md)
- [pdqsort, the Branchless Modern Quicksort](10-pdqsort-the-branchless-modern-quicksort.md)
- [Powersort: Timsort with a Proof](11-powersort-timsort-with-a-proof.md)
- [Radix Sort: Linear Time, No Comparisons](12-radix-sort-linear-time-no-comparisons.md)
- [Counting Sort: When the Key Range Is Tiny](13-counting-sort-when-the-key-range-is-tiny.md)
- [Bitonic Sort and the GPU](14-bitonic-sort-and-the-gpu.md)
- [Live Race: Every Sort, Side by Side](15-live-race-every-sort-side-by-side.md)
- [Engine Case: Sorting Draw Calls](16-engine-case-sorting-draw-calls.md)
- [Engine Case: Sort-and-Sweep Broadphase](17-engine-case-sort-and-sweep-broadphase.md)
- [Picking the Right Sort](18-picking-the-right-sort.md)
- [Pitfalls](19-pitfalls.md)
- [What's Next](20-whats-next.md)
