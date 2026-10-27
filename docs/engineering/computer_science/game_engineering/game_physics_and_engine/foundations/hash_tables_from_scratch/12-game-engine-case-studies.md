# Game Engine Case Studies


Here's what shipping engines and libraries actually build, and why each made the choice it did. The pattern to watch for: when the keys are small integers the engine hands out itself, the fastest "hash table" is often no hash table at all.
1 · Unreal's TMap

TMap<K,V> is implemented as a TSet<TPair<K,V>> backed by a sparse array[20]. Collisions are resolved by chaining rather than probing, but the chain is intrusive. In the engine source each TSet element carries a HashNextId, the sparse-array index of the next element in its bucket, so a lookup hashes to a bucket head and then walks element indices rather than heap nodes. Removal unlinks the element from its bucket's chain and leaves a reusable gap in the sparse array, so no tombstone is needed. The hash function is GetTypeHash(), customizable per key type; the set allocator decides how many hash buckets to use[20]. Epic's documentation calls TMap the most commonly used container in Unreal after TArray[20].
2 · EASTL hash_map

Electronic Arts' EASTL[21] provides hash_map and fixed_hash_map (pre-allocated, no heap allocation during gameplay). The layout is textbook separate chaining: mpBucketArray is an array of node pointers, and every element including the first lives in a heap node holding the value, a next pointer, and optionally a cached hash code so a probe can reject a node without touching the key[22]. Nothing is inlined into the bucket slot. What suits it to fixed memory budgets is fixed_hash_map, which pre-allocates both the bucket array and a fixed node pool, so no allocation happens during gameplay. Note also that EASTL's default rehash policy uses prime bucket counts with a max load factor of 1.0, not the 0.7-to-0.875 range the flat tables in this tutorial use.
3 · Entity-to-component lookup in ECS

In a sparse-set ECS (EnTT and similar), each component type has a dense array of component data and a sparse array mapping entity IDs to indices into the dense array. The sparse array is direct-indexed (not hashed): the entity ID is the array index, giving O(1) lookup with no collision resolution. The archetype engines mostly do not hash entities either. Bevy resolves an entity through a Vec<EntityMeta> indexed directly by the entity's index; Flecs uses a paged sparse-set entity index; Unity DOTS addresses its chunk lookup by entity index. Direct indexing beats hashing here for the same reason the sparse set does: the key space is dense and allocated by you.

When entity IDs do go into a hash table (a map from entity to some sparse per-entity state, say), the hash has to suit the table. Bevy's EntityHasher is a good worked example. For hashbrown's Swiss table it leaves the entity index in the low bits, which pick the slot, so entities spawned together stay together in memory, and it multiplies by a golden-ratio constant to fill the high bits that become H2; an identity hash would leave H2 the same for every small ID, so every occupied slot in a group would match the SIMD filter. Its documentation warns that the scheme suits contiguous indices and that unusual patterns, such as indices that are all multiples of 256, need a general-purpose hash[27].
4 · String interning (FName)

Unreal's FName system hashes the string once, stores it in a global name table (a hash table of unique strings), and thereafter uses the table index as the name's identity. Comparing two FNames is an integer comparison rather than a string comparison. Creating an FName from a string costs one hash and a short probe of the name table.
5 · Shader permutation caches

A material with 12 boolean features generates up to 4096 shader permutations. The compiled-shader cache maps a permutation key (a bitmask or a small struct of flags) to a compiled pipeline state object. The key is small, so a multiplicative hash of the bitmask is enough, and a flat open-addressing table is a natural fit. The lookup sits on the render thread's per-draw path, so it has to stay cheap.

Worked example

12 flags give 212 = 4,096 possible keys, but say a level only uses 300 of them. A 512-slot linear-probing table holds them at α = 300 / 512 ≈ 0.59, under the 0.7 limit from §10.

A successful lookup then averages 0.5 × (1 + 1/(1 − 0.59)) ≈ 1.7 probes, mostly within one or two cache lines.
6 · The flat-map family and its group widths

Three widely used tables share the Swiss-table idea and disagree about the group. Abseil filters 16 slots at a time with SSE2, dropping to 8 where it is unavailable. Boost's unordered_flat_map uses groups of 15 buckets inside a 16-byte metadata word, spending the sixteenth byte on an overflow marker that "removes the need to use tombstones"[15], and pins the maximum load factor at 0.875 with no way for the user to change it[15]. Meta's F14 is named "because it filters 14 slots at once", with a one-byte tag per slot, SSE2 or NEON filtering, and a typical maximum load factor of 12/14[16]. The filtering mechanism is shared; the group width, the load-factor ceiling and the handling of deletions differ.
Implementation: open-addressing hash table
open-addressing hash table
C++
Rust

#include <bit>
#include <cstddef>
#include <cstdint>
#include <vector>

// Minimal open-addressing hash table with linear probing.
// Tombstone deletion. Power-of-2 table size. Fibonacci hash.
struct FlatMap {
    enum SlotState : uint8_t { EMPTY, OCCUPIED, DELETED };

    struct Slot {
        SlotState state = EMPTY;
        uint64_t key;
        uint64_t value;
    };

    std::vector<Slot> slots;
    size_t count = 0;                          // live entries
    size_t tombstones = 0;                     // DELETED slots, still probe-blocking
    size_t mask;                                // table_size - 1

    explicit FlatMap(size_t capacity = 16)
        : slots(capacity), mask(capacity - 1) {}

    // Fibonacci hash: multiply by 2^64 / phi, keep the top log2(size) bits.
    size_t hash(uint64_t key) const {
        // Capacity must be a power of two (and at least 2) for `mask` to be
        // all-ones and the shift to stay below 64.
        return (key * 11400714819323198485ULL) >> (64 - std::countr_zero(mask + 1));
    }

    void insert(uint64_t key, uint64_t value) {
        // Tombstones occupy slots but are not counted in `count`, so the load
        // factor must be tested against occupied + deleted. Testing `count`
        // alone lets a table under insert/erase churn fill entirely with
        // tombstones and never grow, at which point find() never terminates.
        if ((count + tombstones) * 10 > (mask + 1) * 7) rehash();
        size_t idx = hash(key);
        // Walk the whole probe run to EMPTY looking for the key, but remember
        // the first tombstone so we can reuse it. Stopping at the first
        // tombstone would insert a duplicate when the key lives further along.
        size_t firstTombstone = SIZE_MAX;
        while (slots[idx].state != EMPTY) {
            if (slots[idx].state == OCCUPIED && slots[idx].key == key) {
                slots[idx].value = value;             // update existing
                return;
            }
            if (slots[idx].state == DELETED && firstTombstone == SIZE_MAX)
                firstTombstone = idx;
            idx = (idx + 1) & mask;                   // linear probe
        }
        if (firstTombstone != SIZE_MAX) {
            idx = firstTombstone;
            tombstones--;                             // reclaimed
        }
        slots[idx] = { OCCUPIED, key, value };
        count++;
    }

    uint64_t* find(uint64_t key) {
        size_t idx = hash(key);
        while (slots[idx].state != EMPTY) {
            if (slots[idx].state == OCCUPIED && slots[idx].key == key)
                return &slots[idx].value;
            idx = (idx + 1) & mask;
        }
        return nullptr;
    }

    void erase(uint64_t key) {
        size_t idx = hash(key);
        while (slots[idx].state != EMPTY) {
            if (slots[idx].state == OCCUPIED && slots[idx].key == key) {
                slots[idx].state = DELETED;           // tombstone
                count--;
                tombstones++;
                return;
            }
            idx = (idx + 1) & mask;
        }
    }

    void rehash() {
        size_t newSize = (mask + 1) * 2;
        std::vector<Slot> old = std::move(slots);
        slots.assign(newSize, Slot{});
        mask = newSize - 1;
        count = 0;
        tombstones = 0;   // rehashing is what actually reclaims them
        for (auto& slot : old)
            if (slot.state == OCCUPIED) insert(slot.key, slot.value);
    }
};

What's intentionally missing

This implementation skips: iterator support, generic key and value types, heterogeneous lookup (looking up a &str in a String-keyed table), rehashing in place when tombstones rather than live entries fill the table (it always doubles), SIMD metadata (Swiss table optimization), Robin Hood displacement, prefetch hints, and thread safety. Production tables (Abseil, hashbrown, boost::unordered_flat_map) handle most of these; thread safety is left to the caller even there.

[Back to Hash Tables from Scratch](index.md)
