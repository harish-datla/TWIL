# std::map and std::set: Ordered Associative


A replay system stores events keyed by timestamp and needs to play them back in time order, or jump to "everything from 12.5 s onward". That needs a container that keeps its keys sorted as you insert. std::map<K, V> is a sorted, balanced binary search tree from keys to values: each node holds one key, smaller keys go in its left subtree and larger ones in its right. The standard requires logarithmic-time lookup, insert, and erase, and ordered iteration[3]. The standard never names a specific data structure, but every shipping implementation (libstdc++, libc++, MSVC) uses a red-black tree[24]. AVL trees meet the same logarithmic bounds with a shorter tree, but an AVL erase can rotate at every level on the way back up. A red-black insert needs at most two rotations and an erase at most three, with recoloring that is amortized O(1), which fits the standard's requirement that erase(iterator) run in amortized constant time.

Each element is its own heap-allocated node containing the key/value pair, a parent pointer, two child pointers, and a color field. In libstdc++ on x86-64 that header is 32 bytes per node (three pointers plus the color enum padded to pointer alignment), before the payload; the general-purpose allocator adds its own per-allocation header and rounding on top (8 bytes of header on glibc). Lookup is a logarithmic walk from the root, branching by key comparison at each node.

Worked example

A map<int, float> with 1,000 entries on 64-bit libstdc++ and glibc.

Each node is the 32-byte header plus the 8-byte key/value pair, 40 bytes, which glibc serves from a 48-byte chunk: 48 KB for 8 KB of data.

A perfectly balanced tree of 1,000 nodes is 10 levels deep (210 = 1,024). A red-black tree is allowed to be up to 2 × log2(1,001) ≈ 19.9 levels deep. So a lookup visits somewhere between 10 and 19 nodes, each one a separate allocation that may not be in cache.
Live · red-black tree
key 7
insert ▸
find ▸
erase ▸
Reset ⟲
size
15
height
4
last op
···
The tree starts with 15 keys, perfectly balanced at height 4; the readouts give its size, its height and the last operation. Pick a key with the slider and press insert to watch it drop into place and the tree rotate and recolor, or find to highlight the search path. Red-black invariants: every node is red or black; the root is black; the empty (null) leaves count as black; a red node has black children; every root-to-leaf path has the same number of black nodes. These constraints bound the height at 2·log2(n + 1), which keeps lookup logarithmic. Insert and erase recolor and rotate to restore the invariants in O(log n) time[24]. The height stat counts the nodes on the longest root-to-leaf path.
When to use map, and when not

Use std::map when you need ordered iteration (lowest key first, highest key last) or when you need to find ranges by key (lower_bound, upper_bound, equal_range). For example: per-frame replay events keyed by timestamp, where you stream events in time order; or an interval tree by start coordinate.

Don't use std::map as a default associative container. For unordered "find by key" lookups, std::unordered_map is usually faster (constant vs logarithmic on average), and a flat sorted vector (std::flat_map in C++23, or boost::container::flat_map before that) often beats both for read-heavy workloads on small to medium key sets, because the contiguous layout fits the cache[4].
Check Your Understanding Q1 of 1

Your engine stores ~50 audio-effect parameters keyed by name and looked up every frame, never inserted after init. Which is the right container?
std::map: sorted keys, and O(log n) lookup is plenty for 50 entries.
std::flat_map (or a sorted vector of pairs): contiguous keys, binary search, no per-element heap allocations.
std::unordered_map: O(1) average lookup beats any comparison-based search.
0/1 correct

[Back to C++ Containers](index.md)
