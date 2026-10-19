# Swiss Tables (Flat Hash Maps)


Even a well-tuned linear-probing table compares keys one slot at a time, and for string keys each comparison can be a pointer chase. Modern CPUs have SIMD instructions that compare a whole register of bytes at once, and Swiss tables are built to use them. Google's Swiss table design[11] is the one most current high-performance hash maps build on. The core idea: store one metadata byte per slot in a separate array, and use SSE2 instructions (x86's 128-bit SIMD set) to compare 16 metadata bytes in one instruction. A lookup that would take 16 separate comparisons in a conventional table takes one _mm_cmpeq_epi8.
The H1/H2 split

The hash of a key is split into two parts:

    H1 (57 bits in the design notes): picks the probe's starting position[12].
    H2 (the other 7 bits): stored in the metadata byte for that slot. A compact fingerprint used to filter candidates before comparing the full key.

Which 7 bits become H2 has changed between versions. The design notes and early Abseil take H2 from the low bits (H1 = hash >> 7); current Abseil takes the top 7 bits and uses the whole hash, masked by the capacity, for the starting position[13].

Each metadata byte is empty (0x80 = 0b10000000), deleted/tombstone (0xFE), or occupied (0b0xxxxxxx, where the 7 low bits are H2); a fourth value, the sentinel 0xFF, marks the end of the array for iterators. The high bit distinguishes empty/deleted from occupied, and the 7-bit H2 fingerprint filters out roughly 127/128 of non-matching keys before the table compares the full key.
The SIMD probe

A lookup works as follows:

    Compute H1 and H2 from the key's hash.
    Use H1 to find the starting group: 16 consecutive metadata bytes on SSE2, 8 on ARM and the portable fallback, starting at whatever index H1 lands on rather than at an aligned boundary.
    Load the 16 metadata bytes into a 128-bit SSE2 register.
    Broadcast H2 into all 16 lanes of a second register.
    Execute _mm_cmpeq_epi8: each lane that matches produces 0xFF; non-matches produce 0x00.
    Extract a 16-bit bitmask with _mm_movemask_epi8. Each set bit is a candidate slot.
    For each candidate, compare the full key. On a match, return the value.
    If no match, check if any metadata byte in the group was "empty." If so, the key is absent. If the group was full or contained only tombstones, advance to the next group (quadratic probing on groups) and repeat.

Worked example

The widget below uses a key's low 7 bits as its H2, to keep collisions easy to find. Key 32 is 0b0100000, so H2 = 0x20. Key 160 is 0b10100000: drop the top bit and its H2 is also 0x20.

Look up 160 while 32 is in the group: the SIMD compare reports 1 candidate, the full-key compare finds 32 ≠ 160, and the group's empty slot ends the search. One wasted comparison, which a real hash makes rare: about 1 in 128 per occupied slot.

The result: most lookups check one group of 16 slots with a couple of SIMD instructions and zero or one full-key comparison. At Abseil's maximum load factor of 7/8, a 16-slot group has two empty slots on average[13], so most probes end in the first group.
Live · Swiss table SIMD probe
lookup
reset
lookup key 32
H2 matches
0
full-key comparisons
0
result
-
Pick a lookup key and press lookup; the readouts give the H2 matches, the full-key comparisons and the result. The top row is one group's control bytes, with each slot's key underneath. One SIMD compare checks all 16 control bytes against the lookup key's H2 and produces a bitmask of candidates; only those get a full-key comparison. To keep collisions easy to find, this widget's H2 is just the key's low 7 bits: 32 is found in one comparison, 160 shares 32's H2 and shows a false positive (rare with a real hash, about 1 in 128 per occupied slot), and 37 matches nothing and stops at the group's empty slot. With a real hash, most successful lookups need exactly one full-key comparison and most misses need none.
Check Your Understanding Q1 of 2

A Swiss table stores 112 entries across 128 slots (alpha = 0.875). You look up a key whose H2 matches 3 metadata bytes in the first group. How many full-key comparisons does the lookup perform?
None at all, because the SIMD compare against the metadata is already exact.
At most 3, one per H2 match, stopping at the first key that compares equal.
Sixteen, one for every slot the SIMD instruction loaded into the register.
0/2 correct

[Back to Hash Tables from Scratch](index.md)
