# Chaining


Two asset paths hash to the same slot. That's a collision, and with more keys than a handful it's guaranteed to happen, so every hash table needs a rule for it. The simplest rule is to let a slot hold more than one entry. Separate chaining does exactly that: each slot holds a pointer to a linked list of entries that hash to that slot. Insert adds a node to the list. Lookup walks the list comparing keys. Delete removes the node.

The expected list length is the load factor α = n/m. Under the SUHA, an unsuccessful search examines α entries on average; a successful search examines roughly 1 + α/2[3].

Worked example

12 keys in 16 buckets: α = 12 / 16 = 0.75.

A lookup for a key that isn't there walks a whole chain, 0.75 entries on average. A lookup for a key that is there checks 1 + 0.75 / 2 = 1.375 entries on average: the key itself, plus about half of the other entries that share its chain.

Chaining tolerates load factors above 1.0 (more keys than slots). Java's HashMap defaults to a maximum α of 0.75 and doubles its bucket array when it crosses that threshold[18] (std::unordered_map defaults to 1.0), but correctness doesn't depend on staying below 1.0.

The problem is cache behavior. Each lookup dereferences a pointer to the list head, then follows next pointers through heap-allocated nodes scattered across memory. Each pointer chase is a potential DRAM miss (roughly 100 ns on modern hardware[23]). At α = 0.75, the average chain is short, but even one miss per lookup adds up at engine-frame frequency. Open addressing (next section) keeps all entries in a flat array, giving the CPU prefetcher something to work with.

Chaining still ships widely: std::unordered_map, Java's HashMap, Redis's dict and Unreal's TMap all chain. It keeps elements in place (pointers to them survive a rehash), tolerates load factors above 1, and never moves large values. Where none of that matters, open addressing is usually faster on modern hardware.

[Back to Hash Tables from Scratch](index.md)
