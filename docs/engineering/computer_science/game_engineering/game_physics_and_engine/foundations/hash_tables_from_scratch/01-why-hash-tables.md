# Why Hash Tables


The renderer asks for the compiled shader that matches this draw's feature flags, thousands of times a frame. Keep the shaders in a sorted array and each request is a binary search; keep them in a hash table and each request goes almost straight to the right slot. The difference is small for a hundred shaders and large for a million cached entries.

A hash table works by turning the key into a number (the hash) and using that number to pick a slot in an array. A sorted array gives O(log n) lookup via binary search; a hash table gives O(1) expected lookup, insert and delete. On a table too large for the cache, a hash lookup typically costs one or two cache misses (roughly 100 ns each from DRAM), while a binary search over a million keys makes about 20 dependent probes, most of the later ones misses[23]. For lookups an engine makes many times per frame, such as the string interning table (which maps every distinct string to one canonical ID) or the shader permutation cache, that is the difference between a couple of hundred nanoseconds and a couple of microseconds per lookup.

Worked example

1,000 lookups per frame into a table of 1,000,000 keys, at about 100 ns per DRAM miss.

Binary search: log2(1,000,000) ≈ 20 probes. If about half of them miss, that's 10 × 100 ns = 1 µs per lookup, 1 ms per frame.

Hash table: 1 to 2 misses, 100 to 200 ns per lookup, 0.1 to 0.2 ms per frame. A 16.7 ms frame notices the difference.

Hash tables show up wherever you need a key-value mapping with fast access:

    Entity lookups. An ECS maps each entity ID to its component data. Most ECS designs do this with a directly indexed array rather than a hash table, because entity IDs are small, dense integers the engine hands out itself (§12). Hash tables take over when the keys are sparse or come from outside, such as network object IDs or asset GUIDs.
    Asset registries. The content browser maps asset paths (strings) to metadata. String-keyed hash tables, often over interned keys, are the usual structure here.
    Shader variant caches. A shader may compile into thousands of permutations (feature flags, platform toggles). The key is a bitmask or a small struct; the value is a compiled shader object. Lookup happens on the render thread, so it must be fast.
    String interning. The engine maps every unique string to a single canonical pointer. Subsequent comparisons become pointer equality (one instruction) instead of strcmp. Unreal's FName system is built on one.
    Deduplication. Content pipelines deduplicate textures, meshes, and animation curves by content hash. The dedup index is a hash table.


[Back to Hash Tables from Scratch](index.md)
