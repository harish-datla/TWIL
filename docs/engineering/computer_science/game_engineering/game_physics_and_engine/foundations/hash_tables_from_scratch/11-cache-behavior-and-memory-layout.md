# Cache Behavior and Memory Layout


Two tables can do the same number of probes and still differ several times in speed, because a lookup's cost is set by how many cache lines it touches, and several slots can share one line. Open addressing's advantage on modern hardware comes mostly from cache lines. A typical cache line is 64 bytes. A linear-probing table with 8-byte entries fits 8 entries per cache line. A probe that walks 4 consecutive slots likely hits within the same cache line (one miss for the first, then three hits). A chaining table dereferences a pointer to a heap-allocated node, which is almost certainly on a different cache line, so every hop is a miss.

Worked example

16-byte slots (an 8-byte key and an 8-byte value), so 4 slots per 64-byte line.

A linear probe that checks 3 consecutive slots stays inside one line if it starts in the line's first or second slot, which is half the time, and touches two lines otherwise: at most 2 misses.

A chained lookup that visits 3 nodes reads the bucket array and then 3 separately allocated nodes: up to 4 misses.

Swiss tables push this further. The metadata array is separate from the key-value array: 16 metadata bytes fit in a single 16-byte chunk (one quarter of a cache line). Loading and comparing the group metadata is fast because the data is tiny and contiguous. The full-key comparison, which touches the key-value array, usually happens once per successful lookup and rarely on a miss.

Prefetching. Abseil's lookup issues a prefetch for the slots at the probe position before it loads the control bytes, so the slot-array fetch overlaps the SIMD compare instead of starting after a match. It also exposes prefetch(key), which lets code that knows its next key start both fetches early and overlap them with other work[13].

Alignment. It is tempting to assume the group is 16-byte aligned so the SSE2 load is a single aligned load. Abseil does the opposite on purpose: a group starts at whatever index H1 lands on, and the load is _mm_loadu_si128, deliberately unaligned[13]. That is also why the implementation mirrors the first Group::kWidth - 1 control bytes past the end of the array: an unaligned window starting near the end must still be able to read a full group's worth of bytes.

Group width depends on the target. Everything above describes the SSE2 path. Abseil's AArch64 group is 8 slots wide and uses vceq_u8, the 64-bit form, not vceqq_u8; the portable fallback is 8 wide too[13]. Rust's hashbrown makes the same split, taking the SSE2 path because "it allows us to scan 16 buckets at once instead of 8"[14]. Phones and the Nintendo Switch are ARM (PlayStation 5 and Xbox Series consoles are x86), so when reasoning about probe counts on target hardware, check the group width before assuming 16.

[Back to Hash Tables from Scratch](index.md)
