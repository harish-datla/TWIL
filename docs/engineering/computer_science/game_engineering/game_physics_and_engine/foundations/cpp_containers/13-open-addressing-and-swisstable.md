# Open Addressing and SwissTable


Suppose the texture lookup from §12 shows up in a profile. Most of its cost is the chain of pointer hops: bucket array, then node, then maybe another node. Most high-performance hash tables outside the standard library avoid those hops with open addressing: no separate buckets, every entry stored inline in one flat array, and collisions resolved by probing other slots, checking the next slots until the key or an empty slot turns up. The flat layout is the cache argument from §1 applied to hash tables: one DRAM fetch covers several adjacent probe positions. The probing schemes predate the newer tricks that use SIMD instructions, which work on a whole register of values at once. Robin Hood hashing, which cuts probe-length variance by letting keys that have probed farther take slots from keys sitting close to their home slot, dates to the mid-1980s and underpinned several fast open-addressing tables before the SwissTable generation[33].

The most influential modern design is Google's SwissTable[9], open-sourced in Abseil in 2018[10]. Rust's standard HashMap (via the hashbrown crate) and Go's map since version 1.24 use SwissTable designs, and Folly's F14[11] is a close relative. The group size differs: SwissTable scans 16 control bytes at a time, one 128-bit SIMD register, and grows when a large table passes a load factor of 7/8 = 87.5%. F14 packs 14 one-byte tags into a 16-byte chunk header and spends one of the two spare bytes on an overflow count: the number of keys that wanted this chunk but had to be stored further along. Insert increments it and erase decrements it, so an unsuccessful lookup can stop at the first chunk whose count is zero instead of treating deleted slots as tombstones that force it to keep probing[35]. F14's max load factor is 12/14 ≈ 85.7%.

The design has four moving parts:

    The table is one flat array of {key, value} slots plus a parallel array of 1-byte control bytes, one per slot. The control bytes are stored together, separate from the slots, so 16 of them can be loaded at once.
    The hash is split in two: H1 picks the starting position, and H2, 7 bits, goes in the slot's control byte for fast filtering[26]. Seven bits leaves the eighth for tagging the special states (empty, deleted, sentinel), and one byte per slot means 16 control bytes fill one 128-bit SSE register.
    On x86 with SSE2, lookup loads 16 consecutive control bytes (a group) into one register and runs one _mm_cmpeq_epi8 against the H2 being searched for; _mm_movemask_epi8 turns the result into a 16-bit mask of candidate positions[9]. Abseil uses 8-wide groups elsewhere: 64-bit NEON operations on little-endian AArch64, and plain 64-bit integer bit tricks as the portable fallback.
    Each candidate gets a full key comparison. If none matches and the group has an empty slot, the key is absent; otherwise the probe moves to the next group. Most lookups read one group of control bytes and one slot.

Worked example

A SwissTable with 1,024 slots holds up to 7/8 × 1,024 = 896 entries before it grows, and its control bytes take 1,024 bytes, about 1 byte per slot.

How often does the 7-bit filter let a wrong key through? A full slot holding some other key matches the searched H2 by chance with probability 1/128. A group of 16 has at most 15 other keys, so a lookup expects at most 15/128 ≈ 0.12 wasted full-key comparisons per group.

Press "find" on the widget below with a key. H1 picks a group, one compare against the key's 7-bit H2 produces the match mask over its 16 control bytes, and each match gets a full key comparison. A full group with no match sends the probe on to the next group:
Live · SwissTable lookup
lookup key 42
find ▸
insert ▸
seed 30 random ▸
Reset ⟲
size
20 (63% full)
groups
2
last probe
···
overhead
1 B / slot
Each row is one 16-slot group, with every slot's control byte (its 7-bit H2, in hex) above its key; the table starts with about 20 random keys in two groups. Pick a key and press find: the probed group is outlined, control bytes that match the key's H2 light up, and the slot whose key matches turns green. Seed more keys to fill groups and force multi-group probes. The core idea: keep 7 bits of each key's hash in a byte array beside the slots, so one SIMD compare filters 16 slots at once. The same control bytes mark empty and deleted slots, so a similar compare finds where an insert can go. The metadata costs 1 byte per slot, plus the empty slots the 7/8 load factor leaves. A node-based unordered_map instead pays a next pointer per node (often a cached hash too), a bucket array, and an allocator header for every node[7]. For readability the widget uses aligned groups and moves to the next group linearly; Abseil starts a group at any control byte and steps between groups with a triangular (quadratic) sequence.
Where the time goes

On a table that doesn't fit in cache, a closed-addressing std::unordered_map lookup typically pays a miss for the bucket entry, another for the first chain node, and one more for each further node it walks. A SwissTable lookup typically pays one miss for the group's control bytes and one for the matching slot; the SIMD compare itself is a couple of instructions. Google reported 2-3× better performance than std::unordered_map with significant memory savings[9].

No open-addressing hash table has been standardized as of C++26. The practical answer for new engine code is a third-party table: absl::flat_hash_map, boost::unordered_flat_map, or folly::F14FastMap. std::unordered_map stays useful where code relies on its pointer stability or its bucket interface.

[Back to C++ Containers](index.md)
