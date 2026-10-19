# Eviction & persistence

Memory fills as the player explores, so something has to go. Cells leave memory when the player moves away. A streaming-memory counter or a distance or LRU policy decides what goes. The GTA III-era engine kept a counter that rises by each resource's size on load and falls on unload; over the limit, it frees resources nothing is using first, then buildings, preferring ones out of sight and far away[10]. The less obvious question is when to evict.
Load and evict at different radii (hysteresis)

If you load and evict a cell at the same radius, a player pacing back and forth across a boundary loads and frees the same cell every step: the load-evict thrash. Load at radius R and evict at a larger radius R + margin, so a cell has to be clearly gone before it's dropped. It's the streaming version of a Schmitt trigger: an asymmetric threshold that kills oscillation. The §2 widget evicts farther than it loads for exactly this reason.

Worked example

The §2 widget loads within 3 cells of the volume's center and evicts beyond 4.5. A player pacing one cell back and forth moves a cell's distance between, say, 2.8 and 3.8: it loads at 2.8 and is never far enough to be freed. The price is memory. The load circle covers π × 3² ≈ 28 cells, and the band out to 4.5 can hold up to π × (4.5² − 3²) ≈ 35 more, though only cells the player recently passed are actually in it.
A modified cell is a delta over the baseline

The cooked cell on disk is read-only and shared. When the player changes it (opens a door, loots a chest, fells a tree, kills an NPC), you do not rewrite the cooked cell. You record a delta layered on top of the baseline, and that delta is what the save system serializes. This keeps the world's authored data immutable and the per-player mutation tiny. It's the common approach for level-based worlds, not the only one: a single-scene or roguelike game may snapshot whole cells, and an MMO keeps authoritative state on the server.

Worked example

A cell holds 2,000 placed objects. The player opens three doors and loots one chest. Say each change is recorded as an 8-byte object ID plus 8 bytes of new state: the delta is 4 × 16 = 64 bytes, and the cooked cell on disk stays untouched. When the cell streams back in, the loader reads the baseline and then applies those four records.
