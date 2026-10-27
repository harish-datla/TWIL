# Pitfalls and How to Spot Them


Streaming bugs are usually visible. They show up as pop-in, hitches, or "the level took too long to load." The common classes, and how to spot each:
Synchronous I/O on the main thread

Any blocking read(), fopen(), or CreateFile() on the frame thread is a hitch waiting to happen. The median NVMe read returns in tens of microseconds, which fits in a frame; the tail latency is the problem. A cold page cache, a filter-driver stall, a saturated device queue, or an HDD seek turns the same call into multiple milliseconds, and the frame absorbs all of it. The fix is structural: route every read through the streamer, even tiny ones. Audit your codebase for fopen in any frame-path module.
Live · Streaming Hitch
sync read on main thread
async via streamer
Play frames ▶
Reset ⟲

The sync mode periodically stalls the frame thread; the async mode lets the streamer absorb the same latency without the player noticing.
The top strip is the frame thread and the bottom one the streamer's thread, one cell per frame. Press Play frames in each mode: with sync reads the loads land on the frame thread as red hitches, and with async reads the same loads move to the streamer's strip in amber while every frame stays green. A sync read on the frame thread costs whatever the storage stack's worst case happens to be that frame, and one multi-millisecond tail event is enough to drop a frame on a 60 fps target. Faster storage shrinks the typical stall but not the tail, so the fix is to never block the frame thread on storage.
Read amplification

A load that needs 4 KB but reads a whole 64 KB block moves 16 times the bytes it uses, and the other 60 KB are wasted bandwidth unless something else in that block is wanted soon. The fix is to align the asset layout with the read granularity: if reads happen in 64 KB chunks, group assets so each chunk holds data that's loaded together. SVT's tile sizing is the textbook example.
HDD versus SSD assumptions

A game tuned for SSD can be unplayable on HDD. Random-access patterns that work fine on flash collapse under seek time. Marvel's Spider-Man on PS4 budgeted its tile loads against a conservative HDD read speed, since players swap in drives of varying quality[5], and grouped each part of the city's data together on disk to cut seeks, at the cost of storing some objects many times over[7]. If your engine supports HDD installs, test with one and profile with the seek-time penalty present. If it doesn't, say so in the system requirements.
Cache thrash from naïve LRU

A scanning workload (one-time access to many tiles) plus a hot working set (a small number of always-touched tiles) is the worst case for pure LRU: the scan evicts the working set, then the working-set accesses re-evict the scan. The fix is ARC, 2Q, or any policy that distinguishes "scanned once" from "accessed repeatedly." See §8 and the widget there.
Priority bugs cause pop-in

The streamer loads things in priority order, but the priority function might be wrong. Common bugs: forgetting to apply the velocity bonus, weighting screen-space size correctly only when the camera is moving, picking a frustum cone too tight so a quick camera turn leaves you with no resident tiles. Always test with a fast-turning camera and a fast-moving player, and track pop-in events as the proxy metric, as the priority widget in §9 does.
Decompressor starvation

Reads arrive at 8 GB/s. Decompression runs at 1 GB/s. The CPU decompressor's input queue overflows; reads back-pressure; the device idles. This is the classic pre-DirectStorage symptom. Fix it by moving decompression to the GPU (GDeflate, Zstd on GPU), running more decompressor threads, or, as a last resort, using a faster codec at a smaller compression ratio.
Memory fragmentation in the pool

Variable-sized assets in a fixed-size pool will fragment over time. After enough churn, you can't fit a 2 MB texture in 8 MB of free space because the free space is scattered across 16 holes. The two common answers: pool-per-size (allocate from buckets sized to common asset sizes) or pool-per-class (separate pools for textures, meshes, audio, etc.), and many engines combine them.
File handle exhaustion

Some streamers open one file per asset bundle. With 50 bundles loaded that's fine; with 5,000 it can hit a per-process limit (many Linux systems default to 1,024 open descriptors; the Windows C runtime's stdio defaults to 512 open streams) and start returning errors. The fix is to share handles across logical bundles or to use one giant bundle with offset-based requests.
Check Your Understanding Q1 of 2

Your engine runs at a steady 60 fps but every few seconds a single frame runs 8 ms long. Your storage profiler shows occasional ~5 ms read events on the frame thread. What is the most likely cause?
A blocking read on the frame thread. Even an SSD read can take several milliseconds when the cache misses, and any read on the frame thread serializes the frame on it. Route the read through the streamer.
Garbage collection pauses in the scripting runtime, stalling the frame thread at collection boundaries.
The display’s variable refresh window occasionally forces a finished frame to wait for the next scanout.
0/2 correct

[Back to File Streaming for Game Engines](index.md)
