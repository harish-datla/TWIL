# List vs Vector, Measured


The claim that a vector beats a list even where the list "should" win deserves a measurement. Stroustrup's Going Native 2012 benchmark[1] is the standard one: generate N random integers; for each one, find its sorted position (linear scan from begin()) and insert it there. (The original then removes the elements again from random positions.) Run it on both a vector and a list. Both do the same comparisons and the same number of insertions; only the constant factor differs.

Worked example

N = 2,000 (the widget's default). When the sequence holds i elements, a random new value lands in the middle on average, so the search scans about i/2 elements and the vector shifts about i/2 more.

Summed over all 2,000 inserts: 2,000 × 1,999 / 4 ≈ 1,000,000 comparisons for both containers, plus about a million element moves for the vector. Same work on paper. The vector's comparisons read consecutive memory and its moves happen in bulk; the list's comparisons are a million loads that each wait for the one before.

Press RACE to run the insert half in JavaScript. The vector is an Int32Array shifted with copyWithin; the list is a pair of typed arrays (value, next) whose nodes sit at shuffled positions, so following next jumps around memory the way heap nodes do. Absolute times won't match native C++. At these sizes the whole list fits in L1 or L2, so each hop is a cache hit rather than a DRAM miss, and the list's handicap is mostly that every load has to wait for the previous one. Expect a gap of a few times here, widening with N; native runs whose lists outgrow the L3 cache show the order-of-magnitude gap:
Live · sorted-insert race
N elements 2000
RACE ▶
Reset ⟲
vector ms
···
list ms
···
ratio
···
The two bars are the measured times for the same N sorted inserts, scaled so the slower one fills the track. Press RACE, then raise N and race again: the ratio grows with N. At the smallest sizes both runs can finish faster than the browser's timer can measure, so the readouts there are noise. The vector wins because its sorted-position search sweeps contiguous memory, while the list's search is a chain of dependent loads to scattered nodes (a cache miss per node once the list outgrows the cache). The insert phase costs the vector a memmove of cache-resident bytes and the list a pointer relink. The list does less structural work per insert and still loses, because the search dominates. Both are O(n) per insert; the layout sets the constant[1].
When list actually wins

Two cases survive the cache argument. (a) Splice-heavy workloads where you move elements between containers without iteration: an LRU cache that touches one element per request, then moves that node to the front, is a textbook fit. (b) Code that holds long-lived pointers into the container across many mutations: a list<Job> where job dependency edges store raw Job*. Before C++26, the contiguous alternative in both cases needs a separate indirection table to provide the reference stability that list gets for free; whether that's a better deal depends on the specifics. C++26's std::hive (§16) covers case (b) with a cache-friendlier layout when element order doesn't matter, which leaves the splice-heavy case (a) and order-sensitive versions of (b) as the natural homes for list.

[Back to C++ Containers](index.md)
