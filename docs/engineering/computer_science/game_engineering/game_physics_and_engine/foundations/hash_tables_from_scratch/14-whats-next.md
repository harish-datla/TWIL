# What's Next


Next in the series, Sorting Algorithms covers the other half of organizing data: putting it in order, from insertion sort to radix sort, which distributes keys into buckets much like a hash table does. Beyond this page, the hash-table variants worth knowing about:

    Perfect hashing. If the key set is known at compile time (keyword tables, opcode maps), a perfect hash function maps each key to a unique slot with zero collisions. gperf and the CMPH library generate them. O(1) worst case, not just expected.
    Cuckoo hashing. Two tables, two hash functions. Each key has exactly two candidate slots (one per table). Insert displaces the incumbent if both candidates are occupied, kicking it to its alternate slot. Worst-case O(1) lookup (check two slots, done). Resize is rare but O(n) when it happens.
    Concurrent hash maps. Lock-free and lock-striped variants: Java's ConcurrentHashMap, Intel TBB's concurrent_hash_map, folly's AtomicHashMap. The main challenge is resizing without blocking readers.
    GPU hash tables. SlabHash (Ashkiani et al., 2018) and the CUDA hash table in cuCollections. GPU hash tables must tolerate high contention across thousands of threads and avoid warp divergence on probe-chain walks.

Elastic and funnel hashing (2025)

For the engine programmer, the most useful next step is to profile your game's hottest hash table. Measure the load factor, the average probe length, and the cache-miss rate. If the table is chained, benchmark a switch to open addressing. If it's open addressing with a weak hash, try a multiplicative (Fibonacci) mix. If it's already a Swiss table, the remaining wins are usually in the hash function and the key layout. 

[Back to Hash Tables from Scratch](index.md)
