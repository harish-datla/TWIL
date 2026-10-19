# What's Next


The decision comes down to layout (one block, chunked, node-based, hashed) and access pattern; the container follows from those two. The cheat sheet in §19 is the lookup table, and the earlier sections are the reasoning behind each row.

Next in the series, Hash Tables builds a hash table from scratch: hash functions, chaining versus open addressing, load factors and probing, the ideas §12 and §13 used here. Further on:

    The C++ memory model tutorial covers the atomics and orderings that concurrent containers depend on. Bounded lock-free queues are usually ring buffers built on those orderings.
    The job-systems tutorial uses work-stealing deques. They share std::deque's two-ended interface but are usually built on a circular array (the Chase-Lev design): the owning thread pushes and pops at one end while thieves take from the other, which keeps most operations free of contention.
    The Abseil hash containers (flat_hash_map, node_hash_map), boost::unordered_flat_map and Folly's F14 are widely used open-addressing implementations. The F14 design doc lays out the trade-offs between its storage variants[11].
    For production game-engine practice, EASTL's source is worth reading: it reimplements the standard containers around a simpler allocator model and adds fixed-storage and intrusive variants[5].

[Back to C++ Containers](index.md)
