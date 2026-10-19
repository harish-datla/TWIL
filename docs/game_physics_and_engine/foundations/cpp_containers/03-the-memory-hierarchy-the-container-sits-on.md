# The Memory Hierarchy the Container Sits On


Sum a million ints stored in a vector and the loop finishes in well under a millisecond. Sum the same million ints stored in a list whose nodes are scattered across the heap and it can take tens of milliseconds. The adds are identical. What differs is where each int has to come from, and every container is ultimately a way of arranging bytes for this memory hierarchy. On data that doesn't fit in cache, a CPU can spend most of its time waiting:
Tier	Typical latency (cycles)	Capacity	Relative cost of a miss
Register	~0	16-32 general-purpose × 8 B per core	
L1 data cache	~4	32-48 KB per core	
L2 cache	~12	256 KB-1 MB per core	
L3 cache (shared)	~40	4-64 MB per socket	
DRAM	~250	8-128 GB	

Cycle counts are Skylake-class x86 rule-of-thumb numbers, drawn from Hennessy and Patterson[13] and Intel's optimization manual[14]; AMD Zen, Apple M-series, and ARM Neoverse have their own profiles, and server parts with mesh interconnects push L3 and remote-socket DRAM substantially higher. The exact numbers vary by part; the ratio is what matters here. A miss to DRAM costs roughly 60 times an L1 hit on a desktop core.

The unit the cache moves around is a cache line: 64 bytes on x86-64 and most ARM, 128 bytes on Apple M-series. A line holds 16 ints, or 8 doubles, or 8 64-bit pointers. When the CPU touches a byte that isn't in cache, the hardware pulls the entire line that contains it, and pays the DRAM latency once. Every other byte on that line then costs only an L1 hit.

Worked example

Walk 1,000,000 ints, with a DRAM miss at about 250 cycles on a 4 GHz core.

List, if every hop misses all the way to DRAM (likely once the nodes are scattered across more memory than the L3 cache holds): 1,000,000 × 250 = 250 million cycles, about 62 ms. That's almost four 60 fps frames.

Vector: the 4 MB of ints is 62,500 cache lines. Even if every line missed, that's 62,500 × 250 = 15.6 million cycles, about 4 ms, and in practice the hardware prefetcher (below) hides most of those misses.

This is the central fact behind every container choice in the rest of the page. A contiguous container amortizes the cost of one DRAM fetch across 16 elements. A node-based container with a general-purpose allocator usually pays that fetch once per element, because consecutive nodes rarely share a cache line. (Intrusive containers and pool allocators can close some of the gap; see §16 and §17.)

With that established, the containers come in three structural families. The rest of the sections are organized by family:

    Contiguous sequence: array, vector. One block, indexable, cache-loving.
    Chunked or node-based sequence: deque (chunked), list and forward_list (one node per element).
    Associative: map/set (sorted, node-based, tree); unordered_map/unordered_set (hashed).

Plus the adaptors that wrap one of those (stack, queue, priority_queue) and the non-owning views (span, string_view) that point into someone else's storage.

[Back to C++ Containers](index.md)
