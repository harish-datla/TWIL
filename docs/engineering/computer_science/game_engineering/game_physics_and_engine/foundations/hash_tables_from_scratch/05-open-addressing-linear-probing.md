# Open Addressing: Linear Probing


Every node in a chained table is a separate allocation, so every chain hop can be a cache miss. A shader cache on the render thread would rather keep all its entries in one flat array that the CPU can read in order. Open addressing stores all entries directly in the table array. No linked lists, no per-entry allocations, no pointer chasing. When a key hashes to an occupied slot, the table probes forward through the array until it finds an empty slot (for insert) or the target key (for lookup).

Linear probing is the simplest variant: the probe sequence for slot h is h, h+1, h+2, ... (mod m). Insert walks forward until it finds an empty slot. Lookup walks forward until it finds the key or an empty slot (proof that the key isn't present).

The catch is that the fuller the table, the longer the walk. At half full, most keys sit at or next to their home slot. At 90% full, a lookup for a missing key has to walk past long runs of occupied slots before it finds an empty one.

Worked example

A linear-probing table at load factor α = 0.75 (three quarters full).

Successful lookup: 0.5 × (1 + 1/(1 − 0.75)) = 0.5 × (1 + 4) = 2.5 probes on average.

Lookup for a missing key: 0.5 × (1 + 1/(1 − 0.75)2) = 0.5 × (1 + 16) = 8.5 probes. At α = 0.9 the same two formulas give 5.5 and 50.5.

Knuth's analysis[3] gives those expected probe counts under uniform hashing, the same simple-uniform-hashing assumption CLRS chapter 11 works from[25]. The formulas tell you how many slots a lookup checks at a given fullness, which is what decides when the table should grow:
KNUTH §6.4 Unsuccessful:   12(1 + 1(1 - α)2)    Successful:   12(1 + 11 - α)

In words: a lookup that finds its key checks about half of 1 + 1/(1 − α) slots, and one that misses checks about half of 1 + 1/(1 − α)². Both are small while the table is half empty (1.5 and 2.5 probes at α = 0.5) and blow up as α nears 1, the miss cost fastest because of the square. That's why open-addressing tables resize well before α = 1.

The failure mode is primary clustering. When several keys hash to nearby slots, their probe runs merge into a single long cluster. New keys that hash anywhere into the cluster join it, making it longer. A cluster's chance of growing is proportional to its length, so long clusters grow fastest[3]. The widget below makes this visible: as the load factor increases, the table fills with long contiguous runs of occupied slots.
Live · Primary clustering at increasing load
fill to target
reset
target load factor 0.50
slots filled
0 / 64
longest cluster
0
avg probe (insert)
0
The row is a 64-slot linear-probing table. Set a target load factor and press fill to target; the readouts give the slots filled, the longest run and the average probes per insert. Each slot is colored by the length of the run of occupied slots it belongs to, on a fixed scale from blue (a lone key) to red (16 or more). With this widget's keys, the longest run is 11 slots at load factor 0.5 and 13 at 0.7, and at 0.9 a single run covers 44 of the 64 slots: a new key that hashes anywhere into it has to walk to its end. That is primary clustering. Quadratic probing and double hashing break up runs by jumping past neighboring slots.
Live · Probe sequence visualizer
linear
quadratic
double hash
key to insert 42
step time 300 ms
probe ▸
probes taken
5
slot found
2
strategy
linear
The row is a half-full 32-slot table. Pick a key and a strategy and press probe: each checked slot turns red and is numbered, and the slot where the key lands turns green. Linear probing steps by 1 each time: cache-friendly, but clusters merge. Quadratic probing visits offsets 1, 3, 6, 10, ... (triangular numbers) from the home slot, jumping past nearby occupied slots. Double hashing steps by a second hash of the key, forced odd so it visits every slot of this power-of-two table; keys with the same home slot usually follow different sequences. Run the same key under each strategy to compare.
Check Your Understanding Q1 of 2

A linear-probing hash table with 1024 slots holds 900 entries. You look up a key that is not in the table. Roughly how many slots does the lookup examine before it finds an empty slot?
About 5 probes, since the table still has 124 empty slots to run into.
About 35 probes, from the unsuccessful-search formula at this load factor.
About 900 probes, one for every entry currently stored in the table.
0/2 correct

[Back to Hash Tables from Scratch](index.md)
