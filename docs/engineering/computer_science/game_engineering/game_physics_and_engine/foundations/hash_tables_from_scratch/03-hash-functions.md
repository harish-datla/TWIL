# Hash Functions


An asset registry is keyed by paths like textures/rock_01 and textures/rock_02, which differ in one character. If the hash maps them to the same or neighboring slots, every lookup in that corner of the table slows down. The hash function decides how evenly keys spread out.

A hash function maps a key of any size to a fixed-size integer. The table uses that integer, reduced to the table size, as a slot index. Two properties matter:

    Uniformity. The hash should distribute keys uniformly across the output range. Non-uniform hashes pile keys into the same slots, increasing probe lengths.
    The avalanche effect. Flipping one input bit should flip roughly half the output bits. A hash that fails this test (e.g., the identity function on sequential integers) creates patterns in the slot assignments that interact badly with power-of-2 table sizes.

Worked example

64-bit FNV-1a (below) of rock_01 is 0x18F46FB128F0E366 and of rock_02 is 0x18F46EB128F0E1B3.

In a 1,024-slot table (keep the low 10 bits) they land in slots 870 and 435: different, which is what you want. But only 7 of the 64 output bits differ. A hash with full avalanche would flip about 32, which is why FNV-1a is fine for short keys in a table but weak where every output bit matters.
Integer hashing

Division method: h(k) = k mod m. Simple, but the choice of m matters. If m is a power of 2, k mod m is just the low bits of k, and sequential keys land in sequential slots (no mixing). If m is prime, the distribution is better, but prime-modulo division is slow (an integer divide).

Worked example

Keys 16, 32, 48 and 64 in a 16-slot table.

Division method with m = 16: every key is a multiple of 16, so k mod 16 is 0 for all four. One slot, four keys.

Fibonacci hashing (next paragraph), keeping the top 4 bits of k × 2,654,435,769 mod 232: slots 14, 12, 10 and 8. Four keys, four slots.

Fibonacci hashing (multiplication method): h(k) = (k * 2654435769) >> (32 - log2(m)). The magic constant is 232/φ, truncated. The multiply carries every input bit into the higher output bits, and the right shift keeps the top bits, which depend on the most input bits. The Linux kernel's hash_32() and hash_64() use this scheme.
String hashing

FNV-1a[6]: XOR each byte into the hash, then multiply by a prime. One XOR and one multiply per byte: simple and fast on short keys, though its avalanche is weak (xxHash's benchmark table flags FNV64 for poor avalanche[7]). The 64-bit variant uses offset basis 0xcbf29ce484222325 and prime 0x00000100000001B3.

xxHash[7]: XXH64 processes 32 bytes per iteration using four 64-bit accumulators with multiply-rotate-accumulate rounds. Throughput-optimized for bulk data: the project's own reference benchmark puts XXH3 at 31.5 GB/s and XXH64 at 19.4 GB/s on an i7-9700K, against 3.0 GB/s for SipHash. XXH3 takes a 64-bit seed or, separately, a custom secret (192 bytes by default, at least 136). All variants pass SMHasher.

wyhash[8]: very fast on short keys, which suits hash-table use. Uses wide multiplies (64-bit × 64-bit = 128-bit) to mix bits. Go's runtime has used a wyhash-derived function since Go 1.17 on CPUs without the AES instructions its main hash relies on.

For string interning, the hash needs to be fast on short strings, and deterministic across runs (no random seed) if hash values are baked into cooked data or used as persistent IDs. FNV-1a is a common choice. For tables keyed by untrusted input (network-facing code), use a keyed hash with a random per-process key, such as SipHash, to prevent hash flooding. Seeding a fast hash such as wyhash raises the bar, but those hashes aren't designed as keyed PRFs.

[Back to Hash Tables from Scratch](index.md)
