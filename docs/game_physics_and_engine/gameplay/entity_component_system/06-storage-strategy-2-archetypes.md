# Storage strategy 2: Archetypes

A movement system that needs Position and Velocity wants both arrays side by side, entity for entity, with nothing to skip. An archetype groups entities by their exact component set. All entities with exactly {Position, Velocity} live in one table. All entities with {Position, Velocity, Health} live in another. Each table is a set of contiguous arrays, one per component column, plus an entity ID column. Iterating all entities with Position and Velocity means finding every archetype whose component set is a superset of {Position, Velocity} and scanning each matching table sequentially.

This is the storage model Unity DOTS[8], Flecs[15], and Unreal Mass Entity[23] use. The core trade-off vs sparse sets: iteration is a pure sequential scan (the CPU prefetcher's best case), but adding or removing a component moves the entity's data from one archetype table to another. That move is the structural change problem (section 8).

Worked example

An entity in the {Position, Velocity} table (12 + 12 = 24 bytes a row) takes a hit and gains an 8-byte Health. It moves to the {Position, Velocity, Health} table: 24 bytes copied, 8 bytes written, and the old table's last row moved into the hole it left. The movement system's next scan reads both tables front to back without checking anything per entity.
Live · Archetype table visualizer
spawn entity
+ component
- component
reset
archetypes
4
entities
5
selected
e0
Each table is one archetype; the readouts give the archetype count, entity count and the selected entity. Spawn a few entities, then add and remove components on the selected one. Each archetype table stores entities that share exactly the same component set. Click the canvas to cycle through entities. Adding a component to the selected entity moves it to a different archetype (or creates a new one if no archetype with that component set exists yet). Removing a component does the same in reverse. This table-migration cost is the fundamental price of archetype storage.

The two strategies have been compared under controlled conditions rather than by anecdote. Cox et al. built matched implementations and measured both across a range of entity counts. At 50,000 entities the archetype build's median frame latency was 7.410 ms against the sparse set's 13.819 ms; at 10,000 it was 1.721 ms against 2.945 ms (both p < .001). Entity instantiation ran the other way: a median 6,600 ns per entity for archetypes against 1,000 ns for sparse sets (p < .001). At 1,000 entities and below the frame-latency gap disappeared into the noise, reaching no significance at either 1,000 or 100 entities[26]. The paper does not publish its test hardware, so treat the absolute milliseconds as indicative and the direction of each gap as the durable result: archetypes pay at construction and win at iteration. Cross-library numbers settle no ranking, because library, configuration and workload move together: Beimler's ecs_benchmark runs a 1,000,000-entity update across 7 systems on ten library configurations on one stated machine (3.13 GHz, 12 cores, GCC 14.2.1, Linux 6.10.4) and gets 16 ms at the fast end and 102 ms at the slow end, with most configurations between 19 ms and 40 ms. The 102 ms outlier is EnTT's runtime-typed mode, not EnTT itself, which lands at 40 ms and at 35 ms with an owning group. Its README states the caveat this whole subsection rests on: "The results of these benchmarks should be used as a starting point for your own benchmarking efforts."[27]
Check Your Understanding Q1 of 2

A buff system adds and removes a Stunned component on about 2,000 entities per second. Each of those entities carries 9 components totalling 240 bytes. The world uses archetype storage. What does one add cost?
A migration: all 240 bytes move into the archetype holding the same set plus Stunned, and the vacated row is filled by swapping in the table’s last row.
Nothing beyond writing the new value: archetype tables reserve a column for every registered component type, so adding one writes into the row already there.
A pointer update: the row stays where it is, and the archetype simply records that this entity gained a component.
0/2 correct
