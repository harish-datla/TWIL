# Growth, Amortization, and the Cost of push_back


A culling loop calls push_back once per visible entity. Almost every call is cheap, but now and then one call copies the whole list. If that list is a few hundred thousand entries long, the one expensive call can show up as a spike in the frame time. Knowing how a vector grows tells you how often that happens and how to stop it.

When a vector's size() reaches its capacity(), the next push_back has to allocate a bigger block, move every existing element into it, and free the old one. That's a linear-time operation, and growing by a fixed amount each time would make every push linear on average. The standard library avoids this by growing the block by a constant factor (typically 2× or 1.5×), so the cost of all the reallocations across N pushes is O(N) total and the cost per push averages out to O(1)[19].

The widget below records the cost of each push, counting one unit per element written or moved. Reallocations appear as tall red bars; the running average per push settles at a small constant even though the pushes that reallocate cost O(size):
Live · push_back amortization
2× growth
1.5× growth
target size 64
PUSH ▶
Reset ⟲
total work
0
reallocations
0
avg per push
···
Each bar is one push_back, its height the work it did; the green line is the running average. Press PUSH, then switch the growth factor and push again. At the default target of 64, 2× growth reads 127 total, 7 reallocations and 1.98 per push; 1.5× reads 243, 11 and 3.80. With a 2× growth factor (libstdc++ and libc++ default for push_back) the reallocation copies form the series 1 + 2 + 4 + ... + N/2, about N total; add the N element writes and total work is about 2N, so the average cost per push converges toward 2. The 1.5× factor (MSVC, and Folly's fbvector in its middle size range) reallocates more often: the copies sum to about 2N, so the average settles near 3 and swings higher when N lands just after a reallocation. What 1.5× buys is that, after a few growths, the blocks freed earlier add up to enough room for the next allocation, which 2× growth never allows[8].

The equation below adds up all the work for N pushes with 2× growth, for N a power of two and starting from capacity 1: one write per push, plus every element copied at every reallocation. It's the reason push_back is called "amortized O(1)".

Worked example

Push 8 elements with 2× growth. The capacity goes 1 → 2 → 4 → 8, and each step copies the elements already there: 1, then 2, then 4.

Copies: 1 + 2 + 4 = 7. Writes: 8. Total: 15, which is 2 × 8 − 1, or just under 2 units of work per push.
eq. 1 · amortized push_back cost cost(N pushes) = N writes + (1 + 2 + 4 + … + N/2) copies = 2N − 1

In words: N pushes cost N writes plus the copies at each doubling, and the copies add up to less than N, so the total is about 2N. Spread over N pushes, that's a constant amount of work each, which is what amortized O(1) means. Any single push that triggers a reallocation still costs O(size), so code that needs a hard bound on one push has to call reserve() first.
Why 1.5× sometimes beats 2×

A long-lived vector that keeps growing leaves a trail of freed blocks behind it. With a smaller growth factor, those freed blocks eventually add up to enough room for the next, bigger block, so the allocator can hand the same memory back instead of finding fresh address space. With 2× growth that never happens.

Worked example

1.5× growth from a block of 1: the blocks are 1, 1.5, 2.25, 3.375, 5.06. While the live block is 5.06, the four freed ones add up to 1 + 1.5 + 2.25 + 3.375 = 8.125, and the next block needs 5.06 × 1.5 = 7.59. It fits, after four reallocations.

2× growth: the blocks are 1, 2, 4, 8, 16. While the live block is 16, the freed ones add up to 1 + 2 + 4 + 8 = 15, and the next block needs 32. It never fits, because the freed total is always one less than the live block.
Why the cutoff is the golden ratio

That's the case for picking a factor below 2. The case against is that smaller factors mean more reallocations, so the average per-push cost rises. The shipping picks split the difference: MSVC uses 1.5×, Folly's fbvector uses 1.5× in the middle of its size range and 2× at the small and very large ends[8], and libstdc++ and libc++ stay at 2×. Whether the reuse actually happens depends on the allocator: the freed blocks have to be adjacent and coalesced, and nothing else can have claimed them in the meantime.

In practice, if you know how big the vector will get, the cure for all of this is one line:
reserve.cpp · the one line that fixes most push_back perf

std::vector<EntityId> visibleEntities;

// We know roughly how many entities will be visible this frame.
// One allocation up front beats log2(N) reallocations during the loop.
visibleEntities.reserve(estimatedVisibleCount);

for (const Entity& entity : worldEntities) {
  if (frustum.contains(entity.bounds))
    visibleEntities.push_back(entity.id);    // no reallocation until we exceed the reservation
}

Check Your Understanding Q1 of 2

A vector starts empty and you push back 1,000,000 elements. The implementation uses a 2× growth factor. Approximately how many individual element moves happen across all the reallocations combined?
About 20 million (20 reallocations × 1,000,000 elements).
About 1 million, give or take a doubling step.
About 1 billion. Every push could trigger a copy.
0/2 correct

[Back to C++ Containers](index.md)
