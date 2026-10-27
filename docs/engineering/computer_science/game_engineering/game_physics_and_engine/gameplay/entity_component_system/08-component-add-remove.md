# Component add/remove: the structural change problem

A grenade sets twenty enemies on fire, and each one gains a Burning component in the middle of the damage system's loop. In archetype storage, adding a component to an entity means moving its data from the current archetype table to a different one (the archetype that has the old set plus the new component). Removing a component does the same in reverse. Each move copies every component value for that entity. If the entity has 8 components totaling 200 bytes, that is a 200-byte memcpy per structural change.

A single move is cheap. A thousand moves per frame (spawn 500 enemies, each with an add-component-on-spawn pattern) is not: at 200 bytes a move that's 200 KB of copying a frame, before any lookups. The solutions:

    Deferred commands. Systems record structural changes into a command buffer instead of executing them immediately. At the end of the system (or at an explicit sync point), the buffer replays all changes in batch. This avoids invalidating iterators mid-loop and enables batching moves by target archetype. Unity calls this an EntityCommandBuffer. Bevy calls it Commands, documented as "A Command queue to perform structural changes to the World" whose queued commands "are automatically applied in sequence when the ApplyDeferred system runs"[20].
    Archetype edge caching. When entity e moves from archetype A to archetype A+{Health}, the ECS caches the edge "A + Health = B". The next entity that adds Health to the same archetype A skips the archetype lookup and goes straight to B. Flecs[15] stores these edges in a graph connecting archetypes.
    Chunk allocation. Unity DOTS allocates archetype tables in fixed-size chunks (16 KiB). Each chunk holds as many entities as fit. Moving an entity out of a chunk leaves a hole that is filled by swapping in the last entity from the same chunk. This keeps chunks packed without a global compaction pass.

Deferral also fixes a correctness problem. Applying a structural change while a query is iterating the table it reads is a bug, whatever it costs. An archetype table fills a vacated row by moving its last row into the hole, so migrating the entity at index i puts an unvisited entity at index i, and a loop that then advances to i+1 never visits it. Flecs' manual gives the same reason for deferring: "doing this without deferring an operation could modify the underlying data structure."[16]
Live · Immediate vs deferred structural change
Apply mode
immediate
deferred
run
step
new entities
entities visited
0/10
entities skipped
0
migrations
0
bytes copied
0 B
archetype lookups
0
A ten-row source table, the scan cursor, and the destination tables. Run it in immediate mode, then in deferred mode, and compare the migration counts. The system scans the source table and tags entities by health: zero adds Dead, under 40 adds Burning. Five of the ten always qualify. In immediate mode each tag migrates the row on the spot. Swap-and-pop drops the table's last row into the slot the cursor is sitting on (amber), the loop advances anyway, and that row is never visited; at the end of the scan the passed-over rows are outlined in red. The migration count then comes up short of five, which is the tell: entities that should have been tagged were not, so the cheaper-looking run is the broken one. Deferred mode records the same tags into the buffer, finishes the scan having visited all ten, and replays grouped by destination archetype: five migrations, and two archetype lookups instead of one per entity. Per migration the cost is the same either way, 44 bytes, the sum of the four component sizes in the header. Deferral moves the same bytes per entity; what it changes is that every qualifying entity gets moved.

Once both versions actually process every matched entity, deferral does not reduce the copying. The same rows migrate and the same bytes move. They move at a point where nothing is iterating, and grouped by destination archetype, so the archetype-graph lookup runs once per (source, destination) pair instead of once per entity. Playback has to be scheduled somewhere: Unity's EntityCommandBuffer "stores a queue of thread-safe commands which you can add to and later play back,"[9] and its playback resolves the temporary entity IDs that let one command reference an entity an earlier command created. Commands record entity handles rather than row indices for the reason the widget makes visible: by playback time, earlier migrations have already shuffled rows underneath any index you saved.
Check Your Understanding Q1 of 2

A system iterates the archetype table {Pos, Vel, Health} and, for every entity whose health has reached zero, immediately adds a Dead component. The table fills a vacated row by swapping in its last row. What goes wrong?
Migrating row i swaps the last row into slot i, the loop advances to i+1, and that entity is never visited.
The loop reads past the end of the table, because the row count was captured before the loop started.
Nothing, as long as Dead is a zero-sized tag: tag components do not trigger an archetype migration.
0/2 correct

In sparse-set ECS (EnTT), structural changes are cheaper: adding a component inserts into a per-type sparse set (O(1)), removing swaps and pops (O(1)). No table migration. This is the primary advantage of sparse sets over archetypes for workloads with frequent component add/remove (particle systems, buff/debuff stacking, short-lived effects).
