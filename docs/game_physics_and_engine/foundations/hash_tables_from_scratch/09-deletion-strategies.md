# Deletion Strategies


An entity despawns and its entry leaves the table. In an open-addressing table that's harder than it looks. You cannot simply clear a slot, because that breaks probe chains: a lookup for a key whose probe passed through the now-empty slot would stop too early and falsely report "not found."

Worked example

Linear probing. A and B both hash to slot 3, so A sits in 3 and B in 4. C hashes to 4, finds it taken, and sits in 5.

Erase B by marking slot 4 empty. Now look up C: start at 4, find it empty, stop, and report C missing, though it's sitting in slot 5.

With a tombstone in slot 4, the lookup steps past it and finds C. With backward shift, C moves back into slot 4, its home, and nothing is left behind.

Four strategies:

    Tombstones. Mark the deleted slot as "deleted" instead of "empty." Lookups treat tombstones as occupied (skip over them); inserts treat them as empty (reuse them). Simple, but tombstones accumulate and slow down lookups that miss. Swiss tables use this (the 0xFE metadata byte).
    Backward shift deletion. After removing a key, scan forward and shift back any subsequent key whose home slot is at or before the deleted position. Maintains the linear-probing invariant without tombstones. Used by Robin Hood tables (Rust's pre-hashbrown HashMap used this). More expensive per delete, but lookups never encounter stale markers.
    Robin Hood deletion. A variant of backward shift: after removing a key, walk forward; any key with probe distance > 0 shifts back one slot, decrementing its probe distance. Terminates when a key is already at its home slot or an empty slot is found.
    Rehash on delete. After deleting, reinsert every key in the cluster. Correct but expensive. Only viable for tiny tables or when deletions are rare.

Abseil's Swiss table avoids some of those tombstones. Its erase writes kEmpty instead of kDeleted whenever it can prove no probe sequence could have run past the slot. It reads the control group starting at the slot and the group one width earlier, and if the run of non-empty control bytes spanning the slot is shorter than one probe window, the slot goes straight back to empty[26]. That shrinks the tombstone population without eliminating it, which is why tombstones still count against the growth budget: "Tombstones (kDeleted slots) are not included in the growth capacity, because we'd like to rehash when the table is filled with tombstones and/or full slots."[26]
Live · Three erase policies, one op stream
pause
reset
operations
84
tombstone
32.0 / 12 tomb
backward shift
2.4 / 3 moved
group reclaim
6.1 / 5 tomb
empty slots left
0 / 12 / 6
The three rows are 32-slot tables, one per erase policy (blue live, amber tombstone, dark empty), and the chart underneath tracks each one's miss cost as the run plays; the readouts give each table's current miss cost and tombstone or move count. It plays once when the page loads and stops when the tombstone table runs out of empty slots; press reset, then play, to watch it again. Three 32-slot tables, one seeded stream of deletes and inserts, three erase policies. The plotted line is the average probe count for a search that misses, because that is the cost tombstones inflate: a failed search can only stop at an empty slot, and a tombstone is not empty. The naive tombstone table (red) has no compaction, and its miss cost climbs until op 84, when the last empty slot is consumed and every failed lookup walks all 32 slots. Backward shift (green) never creates a tombstone and sits near 2.4 probes for the whole run. Group reclaim (purple) runs Abseil's rule at a 4-slot group width instead of 16, so the run of non-empty bytes fits on screen; it holds the miss cost around 6 rather than 32, and it still leaves tombstones behind, which is why Abseil counts them against the growth budget and rehashes when they pile up.

Tombstone accumulation is a common performance bug in hand-rolled tables. A table with many inserts and deletes fills with tombstones, and lookups that miss scan more and more of them. The fix is periodic compaction (rehash everything into a fresh table, dropping tombstones) or use backward-shift deletion.
Check Your Understanding Q1 of 2

A table with tombstone deletion and no compaction churns for a long time at a constant live-entry count. Which cost grows?
The unsuccessful search, because it can only stop at an empty slot and a tombstone is not one.
The successful search, because every probe now has to test whether the slot is a tombstone.
Memory, because each tombstone holds a copy of the deleted key for the probe to skip.
0/2 correct

[Back to Hash Tables from Scratch](index.md)
