# Why a Container Choice Is a Performance Decision


A particle system keeps its live particles in a std::list, because removing a dead particle from a list is O(1). The profiler says the update loop spends most of its time waiting for memory rather than doing math. Switch to a std::vector, remove dead particles by swapping each with the last one, and the same O(n) loop can run many times faster. The difference is where the elements live.

Big O describes how a data structure's cost grows, not how fast it is. Keeping a linked list sorted and keeping a vector sorted are both O(n) per insert (find the position, then link or shift), and when a memory access cost about as much as an instruction, as it did through the mid-1980s, the two ran at comparable speed. On a modern CPU the vector wins by a margin so wide it stops being an algorithm question and becomes a memory-layout question. Bjarne Stroustrup made this point with a benchmark in his Going Native 2012 keynote[1], and "default to vector" is now the standard advice[16][17]. For sequence containers, the layout in memory is most of the performance story; complexity, reference stability and lookup pattern decide the rest.

The reason is the cache. The CPU doesn't fetch single bytes from memory: it fetches whole 64-byte blocks called cache lines, and keeps recently used ones close by. A vector's elements sit side by side, so one fetched line brings in the next 16 ints for free. A list's nodes sit wherever the allocator put them, so the next node is usually on some other line, and reaching it costs a cache miss: a wait while the line comes from slower memory.

Worked example

The widget below starts with 256 ints. At 4 bytes each that's 1,024 bytes, which is 16 cache lines.

Vector: the sweep enters a new line once every 16 elements, so 16 misses.

List, nodes in shuffled order: each step has only a 15-in-255 chance of landing on the same line as the previous node, so about 1 + 255 × 240/255 ≈ 241 misses, roughly 15 times as many.

The left column reads N integers from a contiguous block (a std::vector<int>). The right column reads N integers from nodes scattered across the heap (a std::list<int>). Both read the same N values, and both traversals are O(n). The difference is the order in which memory gets touched: the vector sweeps through cache lines in order, while the list jumps between unrelated lines and pays a miss on most accesses:
Live · Cache-line walker · vector vs list
vector
list
both
elements 256
speed 2×
RUN ▶
Reset ⟲
vector misses
0
list misses
0
ratio
···
Each cell is one int in memory. Press RUN and watch the vector's head sweep the cells in order while the list's head jumps around; switch to both to see them side by side, and drag the element count to change N. The counters give the misses for each and their ratio: at the default 256, 16 for the vector against roughly 240 for the list. A 64-byte cache line holds 16 ints. The vector's sweep touches a new line once per 16 elements; the list visits the same cells in shuffled order, the way nodes from a general-purpose allocator scatter, so almost every step lands on a new line. A "miss" here is any step onto a different line from the previous one, which models a list too large to stay cached (a list this small would sit in L1 after one pass). The ratio climbs toward 16× as N grows. Real list<int> nodes are 24 bytes rather than 4, and the prefetcher can't help the list, so measured gaps are wider.
Four questions that predict a container's behavior

Every container on this page answers four questions, and you can mostly predict its performance from the answers:

    Where do the elements live? One contiguous block (vector, array), several contiguous blocks (deque, hive), or one independent heap allocation per element (list, map, unordered_map, the node-based containers)?
    How are they ordered? Insertion order (vector, deque, list), sorted by key (map, set), or hashed (unordered_map, unordered_set)?
    Which operations does the container want to be fast? Random index, find-by-key, push at the back, push anywhere, ordered iteration.
    Which references survive which mutations? A pointer into a vector can become invalid on the next push_back; a pointer into a list survives every mutation that doesn't touch that specific node. Engine code that holds long-lived handles depends on knowing this exactly. The rules for when this happens are called iterator invalidation (§7).

Check Your Understanding Q1 of 2

A and B both insert 100,000 ints into a sorted sequence (insert each at its sorted position). A uses a std::vector, B uses a std::list. A runs about 50× faster than B on a modern x86. What is the most accurate diagnosis?
Vector has the better insert algorithm: memmove shifts the tail in one bulk operation, while the list has to relink pointers a node at a time.
The vector’s search for the insertion point sweeps contiguous cache lines; the list pays a cache miss on most node visits. The tail shift is one memmove of cache-resident data.
std::list’s per-node allocation dominates: 100,000 inserts mean 100,000 allocator calls, against the vector’s roughly 17 reallocations.
0/2 correct

[Back to C++ Containers](index.md)
