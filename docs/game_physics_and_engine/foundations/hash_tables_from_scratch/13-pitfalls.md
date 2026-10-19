# Pitfalls


Use this as a review checklist. Each item is a hash table bug that passes ordinary tests and shows up later as a slow frame or a server under attack.

    Hash flooding (DoS). An attacker crafts keys that all hash to the same slot. Expected O(1) becomes O(n). Crosby and Wallach demonstrated this against the Bro intrusion-detection system in 2003, driving it to drop "as much as 71% of its traffic" using less bandwidth than a dialup modem[10]. Defense: use a keyed hash with a random key, such as SipHash. Rust's default HashMap uses SipHash-1-3 with random keys for this reason.
    Identity hashes in Swiss tables. Swiss tables take H2 from the hash bits that don't pick the slot. An identity hash on small integer keys leaves those bits zero, so every occupied control byte in a group matches every lookup and each probe degenerates into full-key comparisons. Mix the high bits in (a multiply is enough) before handing integers to a Swiss table.
    Iterator invalidation. An insert that triggers a rehash invalidates every iterator, and in open-addressing tables it moves every entry, so pointers die too. Erasing the current element inside a range-based for loop over a std::unordered_map is undefined behavior, because the loop then increments an invalidated iterator. Use the iterator that erase() returns to continue.
    Tombstone accumulation. A table with many insert/delete cycles fills with tombstones. Misses and inserts slow down because probes skip tombstones but can't stop at them. Fix: periodic rehash to compact, or use backward-shift deletion.
    Power-of-2 modulo with bad hashes. If h(k) = k and m = 2^p, then h(k) & (m-1) is just the low p bits. If keys share low bits (e.g., pointers aligned to 8 bytes), they all map to the same few slots. Fix: mix the bits before masking (Fibonacci multiply, or h >> 16 ^ h on 32-bit hashes).
    Using mutable fields as hash keys. If you insert a key, then modify the fields the hash depends on, the entry becomes unreachable: the lookup hashes to a different slot than the one the entry occupies. Unreal's documentation calls it undefined behavior for TMap[20].

Check Your Understanding Q1 of 2

Your game server uses a hash table to store active player sessions, keyed by player ID (a 64-bit integer). An attacker sends thousands of requests with crafted player IDs that all hash to slot 0. What happens?
Nothing unusual: the table resolves collisions, so every request is still served in O(1).
Lookups for those keys degrade to O(n), burning CPU and opening a denial-of-service hole.
The table detects the skew during a rehash and rejects the offending requests itself.
0/2 correct

[Back to Hash Tables from Scratch](index.md)
