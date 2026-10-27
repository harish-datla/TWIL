# Beyond std: flat_map, hive, Intrusive Lists, Ring Buffers


Sooner or later an engine hits a case the standard containers handle badly: a table of settings looked up every frame but never changed, an entity pool where objects die constantly while other systems hold pointers to them, a bounded audio queue that must never allocate. The standard's container set is the default, not the ceiling. Here's a short tour of the non-std containers most game engines pick up for those cases:
flat_map and flat_set (C++23 / Boost.Container)

Sorted {key, value} storage wrapped in an associative interface. (boost::container::flat_map is one vector of pairs; std::flat_map keeps keys and values in two separate sorted vectors, so a lookup's binary search touches only keys.) Lookup is O(log n) binary search on contiguous memory; insertion and middle-erase are both O(n) because everything after the change point shifts. Insertion and erase are slower than the tree's O(log n); lookup and iteration are faster; memory overhead is lower. Lookup-heavy, small-to-medium key sets are the sweet spot. C++23 standardized it as std::flat_map and std::flat_set; Boost's version dates to 2004[4].

Worked example

1,000 int → float entries.

std::flat_map: a 4,000-byte vector of keys and a 4,000-byte vector of values, about 8 KB plus any spare capacity, and a lookup's binary search reads only the keys.

std::map on libstdc++ and glibc: 1,000 nodes in 48-byte chunks, 48 KB, as worked out in §11.
std::hive / plf::colony (C++26)

A bucket array with stable references on insert and erase. Elements live in a chain of blocks (each block's capacity is fixed when it's allocated; later blocks are usually larger), and each block has a skipfield recording which slots are erased. Iteration uses the skipfield to jump over runs of erased slots in constant time. Insert reuses an erased slot if one exists, else appends. Erase marks the slot in the skipfield, leaving every pointer to other elements valid. Element order is not preserved: an insert can land anywhere a gap exists[12].

Matt Bentley built it for game-engine object pools, where entities are created and destroyed constantly while other systems hold pointers to live ones[32]. It was adopted as std::hive for C++26.
Intrusive containers (Boost.Intrusive, EASTL)

A list or tree where each element carries the link pointers, and the container only holds the head (and usually the tail and a count); the user manages the storage. Two payoffs: no per-node heap allocation (the link state lives inside the element you already own), and an element can be in several intrusive containers at once, one set of links per container. EASTL's intrusive_list and Boost.Intrusive's list are the best-known implementations[5]. Engine schedulers (wait lists of suspended jobs) and allocator free lists commonly use this pattern.
Ring buffers (custom, no std equivalent)

A fixed-capacity contiguous array with two indices (head and tail) that wrap around. Push and pop are both O(1) with no allocation; the capacity is a hard cap. Used wherever a bounded queue makes sense: audio sample buffers, replay event buffers, bounded lock-free job queues. The standard has no ring buffer, partly because the variants don't reduce to one shape: bounded vs growing, single-producer vs multi-producer, overwrite-when-full vs block-when-full. The concurrent-queue proposal P0260 has been revised about twenty times since 2016 and is not in C++26.
SmallVector / fixed_vector (LLVM, EASTL)

A vector with an inline buffer of N elements; falls back to heap allocation when N is exceeded. The first N pushes touch no heap at all, which matches the workload pattern for "build a small list inside a function and discard it" code. LLVM's llvm::SmallVector<T, N> is the most widely cited implementation[15], and EASTL ships fixed_vector<T, N> in the same shape.

[Back to C++ Containers](index.md)
