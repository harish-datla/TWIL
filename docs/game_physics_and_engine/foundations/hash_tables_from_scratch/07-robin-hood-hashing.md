# Robin Hood Hashing


In a linear-probing table a few unlucky keys can end up ten or more slots from home while most sit at home. The average lookup is fine; the worst ones are slow, and a frame budget cares about the worst ones. Robin Hood hashing[4] evens that out. It is linear probing with one added rule: during insert, if the new key's probe distance exceeds the incumbent key's probe distance at the current slot, swap them. The displaced incumbent continues probing from where the new key took over. The invariant: "steal from the rich (short probe distance), give to the poor (long probe distance)."

For successful searches, the effect is on the spread only. For a fixed key set and hash function, Robin Hood insertion occupies exactly the same slots as plain linear probing: it only changes which key sits in each of them. Total displacement is therefore identical, and so is the mean probe count for a successful search. What shrinks is the spread between the luckiest and unluckiest keys.

Worked example

The widget below inserts 12 keys into 16 slots, crowded around slots 9 to 14.

Plain linear probing: average distance from home 3.42, longest 10.

Robin Hood: same 12 occupied slots, same average 3.42, longest 6. Same total work, but the worst lookup got 4 probes shorter.
What the theory says about the spread

The other win is for lookups that miss, which can stop early. If the resident at the current slot has a smaller probe distance than the search key would have at this position, the key cannot be further along: had it been inserted, it would have taken this slot from the resident. The search stops there instead of running to an empty slot.

Rust's standard HashMap used Robin Hood hashing with backward-shift deletion from 2014 until Rust 1.36 (2019), when it was replaced by the Swiss-table-based hashbrown crate[14]. hashbrown's README reports it as around 2× faster than the implementation it replaced, with 1 byte of overhead per entry instead of 8[14].
Live · Robin Hood insert (steal-from-the-rich)
step
play
reset
step time 500 ms
inserts done
6
swaps this insert
0
max probe dist
1
The row is 16 slots, each showing its key and its distance d from home; the table starts with 6 keys and inserts 6 more. Press step to advance one comparison or play to run them all. The home slot here is key mod 16, with keys chosen to crowd slots 9 to 14 (a real table would use a hash). Each slot shows its key and probe distance d. At each step the key being inserted meets one resident. Red: the resident is at least as far from home and keeps the slot. Amber: the incoming key is farther from home, so they swap and the displaced key carries on probing. Green: the key lands in an empty slot. After all six inserts (12 keys in 16 slots) the longest probe distance is 6. The same keys inserted by plain linear probing end with the same mean distance, 3.42, but a longest distance of 10.
Check Your Understanding Q1 of 2

In Robin Hood hashing, when can a lookup for key K terminate early (before finding an empty slot)?
When the slot holds a key whose hash value is larger than K’s hash value.
When the slot holds a key whose probe distance is less than K’s would be here.
When the slot holds a key that was inserted more recently than K was.
0/2 correct

[Back to Hash Tables from Scratch](index.md)
