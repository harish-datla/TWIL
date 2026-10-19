# Powersort: Timsort with a Proof


Timsort finds the runs, but then has to decide which runs to merge first, and a bad order can do noticeably more copying than necessary. Munro and Wild's 2018 paper "Nearly-Optimal Mergesorts"[15] attacked the part of timsort that even Tim Peters described as a heuristic: the merge policy. Timsort's stack-invariant check is a pragmatic rule that usually produces balanced merges, with no optimality guarantee. Powersort replaces it with a rule derived from Mehlhorn's nearly optimal binary-search-tree construction: give each boundary between adjacent runs a power based on where it falls in the array, and merge pending runs whose boundary power is greater than that of the newest boundary.

The result is a stable mergesort whose total merge cost is provably within an additive O(n) of the optimum for the given runs (measured by the entropy of the run lengths). Timsort's policy has no such guarantee and can be measurably worse on some run-length patterns; on typical inputs the two are close. Tim Peters merged powersort into CPython in September 2021 (gh-78742 / PR #28108), shipping in Python 3.11 in October 2022[23]. The patch kept the rest of timsort (run detection, minrun, binary insertion sort, galloping) intact and swapped only the merge-trigger rule. PyPy, Apple's WebKit, and AssemblyScript also use powersort.

Worked example

An 8-element array with a run of 3 followed by a run of 5. Their midpoints are at 1.5 and 5.5, or 0.1875 and 0.6875 of the array. In binary those are 0.0011 and 0.1011, which differ in the first bit after the point, so the boundary has power 1: it sits at the root of a balanced merge tree and is merged last.

Two runs of 2 at the start of the same array have midpoints 1/8 and 3/8, binary 0.001 and 0.011, which first differ in bit 2: power 2, one level down, merged earlier.

For a new run-adaptive stable sort, the powersort policy is a sensible default. Many existing timsort ports (Java's object sort among them) still use the original heuristic, because on typical inputs the difference is small.


[Back to Sorting Algorithms](index.md)
