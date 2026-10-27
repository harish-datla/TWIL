# Residency Pools and Eviction Policies


The player runs laps around a market square, and every lap the streamer reloads the same stall textures, because one quick look at the skyline pushed them out of memory. Which asset to throw out is a policy decision, and the wrong policy turns into extra reads. Once an asset is loaded, it sits in RAM (or VRAM) until something kicks it out. The data structure that tracks who is in and who is out is a residency pool: a fixed-size cache of assets, indexed by ID, with an eviction policy. A large engine usually has several (a texture pool, a mesh pool, an audio pool, an animation pool), each with its own budget. On PC, AMD's Radeon Memory Visualizer[45] shows the GPU side of this in a captured trace: allocations, how the driver and OS back them with physical memory, and paging when a heap is oversubscribed.

The policy question is which asset to evict when a new one needs space. Four classical choices:

    LRU (Least Recently Used). Evict whatever has gone longest without being used. A doubly-linked list ordered by last access: on a hit, move to the front; on a miss, evict from the back. It's the textbook default.
    LFU (Least Frequently Used). Track an access counter per asset; evict the lowest. Adapts to "frequency" rather than "recency." Vulnerable to a one-time spike permanently inflating a counter.
    Hot/cold (2Q, segmented LRU). Two lists with a promotion rule; 2Q is Johnson and Shasha's 1994 version[43]. New items enter the cold list and only move to the hot list on a second access; eviction always pulls from cold first. Scan-resistant by construction: a one-shot sweep over more items than fit in cache churns only the cold list, leaving the hot working set untouched. Linux's active/inactive page lists and Caffeine's W-TinyLFU (TinyLFU admission filter in front of an SLRU) are production variants.
    ARC (Adaptive Replacement Cache). Megiddo and Modha's 2003 paper at the FAST storage-systems conference[27]. Maintains two LRU lists (one for recent items, one for frequently-accessed items), plus two "ghost" lists of evicted IDs. Ghost-list hits tell ARC which way to shift its split. Adaptive and low-overhead; ZFS's cache is a modified ARC.

Worked example

A texture pool sees 1,000 requests a second. At a 90% hit rate, 100 of them miss and become disk reads. At 50%, 500 do: five times the read traffic from the same pool size, purely because of what the policy chose to keep.

The widget compares LRU and ARC on the same request stream: a working set of 8 hot items (green) mixed with requests spread across 200 cold items (purple), most of which are touched once. The mix slider sets the share of requests that go to the working set:
Live · LRU vs ARC
LRU
ARC
cache size 16
working-set / scan mix 0.50
Run ▶
Pause ⏸
Reset ⟲
requests
0
hits
0
hit rate
···
The row of slots is the cache: green items are the working set, purple ones the one-time scan. Press Run with LRU, then switch to ARC; both replay the same seeded request stream. The hit rate controls bandwidth pressure on the streamer. A 90% hit rate means only 10% of accesses turn into reads; a 50% hit rate means five times as many. At the default settings ARC settles near a 50% hit rate against LRU's 40%: ARC serves almost every working-set request from cache, while LRU loses about a quarter of them to cold items pushing hot ones out. For a given cache size and workload, the eviction policy sets that ratio.
Pin lists and priority tiers

Pure LRU/ARC is rarely shipped raw. Production pools layer two things on top:

    Pin lists. Some assets must never be evicted: the player character mesh, the UI atlas, the loaded weapon textures. They're "pinned" and don't participate in eviction.
    Priority tiers. Different asset classes get different effective recency. A texture used by a UI screen is "more important" than a texture on a distant rock. The eviction ranking is tier × recency, not raw recency.

Heat tracking

An alternative to discrete tiers is continuous "heat": every access bumps the heat by a constant; heat decays exponentially over time. The eviction candidate is simply the lowest-heat item. It's a clean abstraction that subsumes both recency (because heat decays) and frequency (because heat accumulates).

[Back to File Streaming for Game Engines](index.md)
