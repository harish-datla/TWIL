# std::list and std::forward_list


A job scheduler moves jobs between a ready queue and a waiting queue, and other systems hold pointers to those jobs the whole time. Moving a job must not copy it, and it must not invalidate anyone's pointer. That is the job std::list is built for. It's a doubly-linked list: each element lives in its own heap allocation, with two pointers (prev and next) to its neighbors. std::forward_list drops the prev pointer to halve the per-node overhead, at the cost of being singly-linked[23].

list has two real selling points in modern C++:

    Reference stability under all mutations. A pointer into a node survives every insert and every erase that doesn't touch that specific node. That matters in code that keeps long-lived handles to objects in containers.
    O(1) splice. list1.splice(pos, list2, it) moves a node from list2 into list1 by relinking pointers, with no copies and no allocations[23]. Apart from forward_list::splice_after, no other standard container moves an element between containers in O(1); C++17's extract/insert moves map and set nodes without copying, but the insert still pays for the lookup. LRU-eviction caches and ready-queue schedulers both lean on this.

Outside of those two use cases, it is almost always the wrong choice. The cache-miss penalty from §1 is the headline; smaller costs add up too:

    Per-node allocation. Each push_back on a list<int> calls the allocator. A 2×-growth vector calls it about log2(N) times.
    Per-node memory overhead. On libstdc++ x86-64, a list<int> node struct is 24 bytes (8-byte prev, 8-byte next, 4-byte int, 4 bytes of padding). glibc's malloc serves a 24-byte request from a 32-byte chunk (the extra 8 bytes are its size header), so each 4-byte int costs 32 bytes of heap: 8× the payload, before any fragmentation.
    No random access. l[i] doesn't compile; you walk from begin().

Worked example

100,000 ints on 64-bit libstdc++ with glibc's malloc.

vector<int>: 100,000 × 4 = 400 KB in one block, one allocation if you reserve first.

list<int>: 100,000 nodes, each a 32-byte heap chunk, so 3.2 MB and 100,000 allocator calls. Eight times the memory, spread over eight times as many cache lines.

The next section measures the gap directly.

[Back to C++ Containers](index.md)
