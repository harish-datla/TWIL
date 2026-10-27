# Quadratic Probing and Double Hashing


The clustering widget showed one run swallowing 44 of 64 slots at load factor 0.9. Linear probing's clustering problem comes from the step size: +1 every time, so any key that lands anywhere in a run extends it. The fix is to step further on each probe, so that keys starting at nearby slots fan out instead of queuing behind each other. Two mitigations:

Quadratic probing: the probe sequence is h, h+1, h+3, h+6, h+10, ... (adding 1, 2, 3, 4, ... to the previous offset, i.e., triangular numbers). This breaks primary clustering because keys that hash to different slots follow different probe paths. It introduces a weaker problem called secondary clustering: keys that hash to the same slot still follow the same sequence, but keys at different slots don't pile up.

There is a coverage guarantee: if the table size m is a power of 2, the triangular offsets 0, 1, 3, 6, ..., m(m−1)/2 are all distinct mod m, so the sequence visits every slot exactly once before repeating.

Worked example

An 8-slot table. The first eight triangular offsets are 0, 1, 3, 6, 10, 15, 21, 28.

Mod 8 they are 0, 1, 3, 6, 2, 7, 5, 4: all eight slots, each exactly once. A quadratic probe can't loop forever while an empty slot exists.
How Python's dict probes

Double hashing: the step size is a second hash function h2(key). The probe sequence is h1(key), h1(key) + h2(key), h1(key) + 2·h2(key), ... The step size must share no factor with m (with a power-of-2 table, force h2 odd) so the sequence reaches every slot. Because the step depends on the key, keys with the same home slot usually follow different sequences, which removes both primary and secondary clustering. With a good second hash the expected probe counts come close to the uniform-probing ideal: 1/(1 - α) for unsuccessful search, (1/α) ln(1/(1 - α)) for successful search. At α = 0.75 that's 4 probes for a miss and about 1.85 for a hit, against linear probing's 8.5 and 2.5.

The downside: double hashing's non-sequential probe pattern defeats the CPU prefetcher. Linear probing's sequential scan loads the next cache line automatically; double hashing jumps around the array, incurring a cache miss on most probes. In practice, linear probing with a good hash function and a moderate load factor (α ≤ 0.7) often beats double hashing on wall-clock time despite having higher probe counts, because most of its probes land on a cache line that is already loaded.

[Back to Hash Tables from Scratch](index.md)
