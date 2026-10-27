# Timsort, the Natural-Runs Mergesort


Last frame's sorted event list gets a handful of new events appended, and now needs sorting again. Almost all of it is already in order. A sort that notices that can finish in close to one pass instead of a full n log n. Tim Peters started writing timsort for Python's list.sort in 2002. The design document, Objects/listsort.txt in the CPython repo, is a detailed record of the design decisions and their measurements, and worth reading in full[11]. The premise: real data is often not random. Logs arrive mostly sorted, with a few out-of-order entries. Database query results arrive in chunks that are themselves sorted. Game-engine event streams from the prior frame are usually monotonic in timestamp. A sort that exploits existing order can beat one that ignores it.

Timsort is a stable mergesort tuned for runs. The four moving parts:

    Run detection. Scan left-to-right looking for the longest stretch of consecutive elements that is non-decreasing or strictly decreasing. Reverse the decreasing runs in place (so they become increasing). A "run" is the unit of work.
    Minimum run length (minrun). If the natural run is shorter than minrun, extend it to minrun elements with binary insertion sort. CPython picks minrun between 32 and 64 so that n / minrun is a power of two or slightly less, which keeps the merges balanced; arrays under 64 elements are just binary-insertion-sorted whole[11].
    The merge stack. Push each run onto a stack. After each push, check two Fibonacci-style invariants on the top three runs (X, Y, Z, top of stack last): runlen(X) > runlen(Y) + runlen(Z) and runlen(Y) > runlen(Z). If either is violated, merge Y with the smaller of X and Z, and repeat. This keeps merges roughly balanced and the stack short.
    Galloping merge. When one run keeps winning during a merge (a long stretch of its elements all precede the other run's next element), timsort switches to exponential search to find where the streak ends, so a streak of k wins costs O(log k) comparisons instead of k.

Worked example

Run detection on [1, 2, 3, 9, 8, 7, 4, 5, 6]: 1, 2, 3, 9 is non-decreasing, and 8 breaks it. 8, 7, 4 is strictly decreasing, and 5 breaks it, so it's reversed in place to 4, 7, 8. Then 5, 6. Three runs, merged pairwise.

minrun for n = 2,112: CPython keeps the top 6 bits of n (2,112 = 33 × 64, so they read 33) and adds 1 if any lower bit is set (none is). minrun = 33, and 2,112 / 33 = 64 runs, a power of two, so every merge is between runs of equal length.
Live · Timsort run detection and merge stack
input shape
Run ▶
Reset
runs found
···
comparisons
···
vs n log n
···
The top half is the array, with each detected run colored (green ascending, orange descending and then reversed), and the bottom half is the merge stack. Pick an input shape and press Run; the readouts give runs found and comparisons. Mostly-sorted input reads 12 runs and 242 comparisons, already-sorted input 1 run and 55. The widget works on 56 elements and skips minrun (real timsort would binary-insertion-sort an array this short in one go) so the natural runs and the merge policy stay visible. "Comparisons" counts the run-detection comparisons plus, for each merge, the combined length of the two runs (an upper bound on the merge's comparisons; galloping is not modeled). On already-sorted input timsort runs in O(n): one run found, no merges. On random input it costs about as much as a plain stable mergesort. Its benefit is on data with existing order: the measurements in listsort.txt show near-linear times on ascending, descending, all-equal, and ascending-with-a-few-changes inputs, where a plain mergesort still pays n log n[11]. The merge stack height is bounded by about logφ(n) ≈ 1.44 log2n when the Fibonacci-style invariant holds; when it doesn't, the stack can outgrow its fixed allocation, which is the 2015 bug below.
The 2015 invariant bug

Stijn de Gouw and colleagues at CWI Amsterdam, using the KeY verification tool (which proves, with human guidance, that Java code meets a formal specification) to prove timsort correct, discovered that the merge stack's invariant was not actually being maintained[14]. mergeCollapse only checked the invariant against the top three runs of the stack. After it merged the top runs and pushed the result back, runs further down the stack could be left in a state that violated the invariant globally. The error did not produce wrong output, but it broke the bound on stack depth: de Gouw's team constructed run-length sequences that overflowed the fixed-size run stack OpenJDK had allocated, so sort() threw ArrayIndexOutOfBoundsException.

Two fixes were available: enlarge the stack to cover the actual worst case under the broken invariant, or fix mergeCollapse so the invariant really holds, as de Gouw's paper proposes. OpenJDK took the first path and enlarged the stack. CPython took the second: its merge_collapse now also checks the fourth run from the top (bpo-23515, committed February 2015, released in 2.7.10 and 3.4.4).

Formal verification of a standard-library sort was practical by 2015, and it found a bug that thirteen years of production use and testing had not. The inner loops of sorts get exercised by every test; the bookkeeping around them (stack invariants, merge ordering, recursion bounds) is read by few and proved by fewer.

[Back to Sorting Algorithms](index.md)
