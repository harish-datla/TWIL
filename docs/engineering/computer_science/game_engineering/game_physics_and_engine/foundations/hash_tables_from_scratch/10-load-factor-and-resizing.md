# Load Factor and Resizing


A table that never grows gets slower with every insert, as §5's formulas showed. A table that grows has to move every entry when it does, and on a game server that one big move can cost a frame. Both the threshold and the way the move happens matter. When the load factor exceeds a threshold, the table allocates a larger array and reinserts every key. The threshold depends on the probing strategy:

    Linear probing: resize at α = 0.5 to 0.7. Higher than 0.7 and clustering dominates.
    Robin Hood: can run near α = 0.9 (Rust's pre-2019 HashMap used 10/11) because probe lengths stay tightly grouped, though inserts and misses still slow down.
    Swiss tables (Abseil): resize at α = 7/8 = 0.875. The SIMD probe tolerates high load because most lookups still check only one group.
    Chaining: resize at α = 0.75 (Java's default) or 1.0 (std::unordered_map's). Can exceed 1.0, with cost growing linearly in α.

Worked example

Java's HashMap starts with 16 buckets and a 0.75 threshold: 16 × 0.75 = 12. The 13th insert pushes it past, and it doubles to 32 buckets (next threshold 24).

A linear-probing table that resizes at 0.7 with 1,024 slots grows when the entry count passes 1,024 × 0.7 = 716.8, so on the 717th insert.
Growth policies

Power-of-2 growth (double the table size): modulo reduces to a bitmask (h & (m-1)), avoiding integer division. Requires a hash function with good avalanche; a bad hash combined with power-of-2 modulo creates systematic collisions. Most open-addressing tables use this.

Prime growth (a prime near double the old size): better distribution with mediocre hash functions, but the modulo is an integer division, which costs tens of cycles on many x86 cores unless the compiler knows the divisor at compile time and can turn it into a multiply. libstdc++'s and libc++'s std::unordered_map use prime bucket counts.
Incremental rehashing (the Redis approach)

A full rehash of n keys is O(n). For a real-time system (a game server, an in-memory database), a single O(n) stall can be unacceptable. Redis[19], whose dict is a chained table, avoids it with incremental rehashing: during a resize it keeps two tables, ht[0] and ht[1]. The new table is allocated up front, but entries move over one bucket at a time as regular inserts, deletes and lookups run (plus a little on a timer). New keys go straight into ht[1]. A lookup checks ht[0] unless the key's bucket there has already been migrated, then ht[1]. The field rehashidx tracks progress; when it reaches the end, the old table is freed.
Live · Stop-the-world rehash vs incremental rehash
stop-the-world
incremental
pause
operations
120 / 120
worst op (touches)
65
average op (touches)
2.4
state
single table (64 buckets)
The two rows of buckets are the old and new tables, with an amber cursor marking the migration point in incremental mode, and the bars below are the cost of each recent operation. Switch between the two modes and compare the worst-operation readout. Both modes run the same 120-operation mix (inserts and lookups, about half each) against a chained table that starts with 16 buckets and doubles whenever the entry count reaches the bucket count, so it grows twice, to 32 and then 64. Cost counts memory touches: one per bucket read and one per chain node visited or moved. Stop-the-world moves every entry inside the insert that triggers the resize: its worst operation makes 65 touches against a 2.4 average, and that bar is the frame drop. Incremental allocates ht[1], sets rehashidx to zero, and migrates one bucket per later operation (skipping at most ten empty ones on the way), so its worst operation makes 5 touches, at the same 2.4 average. Total work is nearly identical; only its distribution changes. While both tables exist, a lookup whose old bucket hasn't moved yet checks both tables.

Amortized O(1) and a bounded worst case are different promises, and the widget separates them. Total work over the run is close either way. What the stop-the-world table cannot offer is a ceiling on any single operation, and a frame budget constrains the worst case.
Amortized O(1) proof sketch

Each insert costs O(1) expected. A resize from n to 2n slots costs O(n) but happens only after about n/2 inserts since the previous one (the table grew from n/2 entries to n). Spread that cost over those inserts and each pays O(1) amortized, meaning averaged over the whole run of inserts.
The accounting version

Live · Load factor vs average probe length
all
linear
uniform (ideal)
chaining
highlight load factor 0.70
linear (successful)
2.17
uniform (successful)
1.72
chaining (successful)
1.35
Drag the highlight to a load factor and read each scheme's average probes for a successful lookup: at the default 0.70, linear probing reads 2.17, uniform probing 1.72 and chaining 1.35. Linear probing (red) climbs steeply past alpha = 0.75: its successful-search cost grows as 1/(1 − alpha), and misses as 1/(1 − alpha)². The uniform/ideal curve (blue, the uniform-probing formula that double hashing approximates) rises more gently. Chaining (green) is a straight line: 1 + alpha/2. Robin Hood has the same expected probe count as linear probing for successful searches, with a much smaller spread. Swiss tables can run at alpha = 0.875 because one SIMD compare checks 16 slots at once, so per-slot probe counts like these overstate their cost.
Check Your Understanding Q1 of 2

A 60 fps game server rehashes a large table on a single insert and drops a frame. A colleague objects that hash insertion is amortized O(1), so this cannot be the cause. What is the flaw?
Amortized O(1) bounds the average over a run of operations, not any single one.
Amortized O(1) assumes a good hash function, and a poor one makes inserts O(n).
Amortized O(1) is for lookups only; insertion has always been O(log n) on resize.
0/2 correct

[Back to Hash Tables from Scratch](index.md)
