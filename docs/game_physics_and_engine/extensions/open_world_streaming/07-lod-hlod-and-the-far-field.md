# LOD, HLOD & the far field

A cell beyond the load radius still has to be seen if it's on the horizon. Rendering it at full detail blows the budget; showing nothing leaves a hole. The answer is detail that falls off with distance: full meshes near, lower LOD at mid range, and a single baked HLOD proxy for the unloaded far cells. Unreal builds HLOD proxies offline (instanced, merged or simplified meshes) to stand in for unloaded World Partition cells and cut draw calls in the distance[11].

Worked example

With the player at the center of the widget's 18 × 10 grid: 16 full cells at 100k triangles are 1.6M, 44 LOD cells at 28k are 1.23M, and 120 far cells as 4k proxies are 0.48M, about 3.3M in all, under the 5M budget. Without HLOD those 120 far cells cost 100k each, 12M on their own, and the total is 14.8M, about three times the budget.

The widget shows concentric rings of detail around the player, with illustrative per-cell triangle counts. Near cells are full meshes, mid cells drop to a cheaper LOD, far cells collapse to an HLOD proxy. Toggle HLOD off and the far field reverts to full meshes and blows the triangle budget; the other alternative is an empty horizon that pops in as you approach:
HLOD proxy for far cells
Watch the rings of detail move with the player, then untick HLOD proxy for far cells. Green is full mesh near the player (100k triangles per cell), amber is a cheaper LOD at mid range (28k), blue is covered by HLOD proxies (4k); the readout compares the scene total with a 5M budget. With HLOD on, the far field costs little and the total stays under budget. Turn it off and the far cells revert to full meshes: the total grows about five times over and turns red.

Texture detail streams on the same principle one level down: pick a mip from the on-screen texel-to-pixel ratio and the memory budget, dropping one mip at a time when the plan is over budget[4]. Sparse virtual texturing takes that further, mapping a small physical cache to a huge logical texture through a page-table indirection. Many implementations decide which pages to stream from GPU feedback (a readback of which pages the visible pixels needed); clipmaps, an older scheme that suits terrain, instead keep a window of each mip level centered on the viewpoint and need no feedback pass[12]. The File Streaming tutorial derives SVT and the I/O fast path in full. Texture streaming is a related but distinct system from world-cell streaming.
Pop-in and hitching are different bugs with different fixes

Pop-in is data that hasn't arrived: a low-detail proxy (or nothing) shows, then swaps to full detail once it streams in. Its causes are throughput, priority and bandwidth; its fixes are more bandwidth, better priority, a larger prefetch radius, or an impostor to cover the gap. Hitching is a single frame that blew its budget, from main-thread integration or GC (§5); its fix is time-slicing. The platform I/O fast path, DirectStorage with GPU decompression on PC and the PS5's hardware decompressor, attacks bytes-to-memory latency, so it helps pop-in and load times, not the integration hitch[13]. A faster SSD will not fix a stutter that lives on the main thread.
Check Your UnderstandingQ1 of 3

Your open-world game stutters each time the player crosses into a new region, even on a fast NVMe SSD. The most likely cause and fix?
The main-thread integration work (registering components, building GPU resources, adding scene primitives) lands on one frame. Time-slice it across frames under a per-frame budget.
The SSD is too slow; switch to DirectStorage.
The cells are too small, so too many load at once.
0/3 correct

Wrong answers, and why: a per-region stutter on an SSD is main-thread integration (time-slice it), not the disk or cell size; a blob-then-detail swap is pop-in (throughput), not a hitch or GC; and boundary thrash is fixed by asymmetric load/evict radii (hysteresis), not by resizing cells or never evicting.
