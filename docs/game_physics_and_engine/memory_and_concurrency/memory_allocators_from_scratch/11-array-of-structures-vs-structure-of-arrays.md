# Array-of-Structures vs Structure-of-Arrays


A wind pass nudges the position of every particle each frame and reads nothing else, yet with particles stored as whole structs it pulls every particle's velocity, health and flags through the cache too. Allocators decide where memory comes from. Layout decides how it's arranged, and on the same CPU running the same algorithm, layout often decides whether the core is fed or stalled on memory. The headline experiment is Tony Albrecht's "Pitfalls of Object-Oriented Programming" from 2009[21]. On a PS3 scene-tree traversal of 11,111 nodes, he reports four data points: 19.6 ms for the original OO graph of node objects, 12.9 ms after custom allocators place the nodes, matrices, and bounding spheres contiguously ("35% faster just by moving things around in memory!", per the slide), 4.8 ms after restructuring the traversal to process that data in order with the hierarchy made implicit, and 3.3 ms after adding software prefetch on top. About 6× end-to-end, of which roughly 4× comes from layout and traversal order and the remainder from prefetching.

The difference is a cache question. With the 32-byte Entity below, a 64-byte cache line holds two whole entities in AoS, so a loop that reads only the 12-byte position uses 24 of every 64 bytes it pulls in. In SoA the same line holds 16 floats of just the field the loop reads, so every byte fetched is used.

Worked example

128 entities of 32 bytes, and a loop that reads only the 12-byte position.

AoS: the entities fill 128 × 32 / 64 = 64 cache lines, and the loop has to load all of them. It uses 12 of every 32 bytes: 37.5%.

SoA: the positions alone fill 128 × 12 / 64 = 24 lines, every byte used. That's 64 / 24 ≈ 2.7× fewer lines for the same work.

Read position and velocity and SoA loads 48 lines against AoS's 64; read every field and both load 64.
Array-of-Structures (AoS)

struct Entity {
  vec3 position;   // 12 B
  vec3 velocity;   // 12 B
  float health;    // 4 B
  uint32_t flags;  // 4 B
}; // 32 B per entity

Entity entities[N];

// Loop touches positions only:
for (int i = 0; i < N; ++i)
  entities[i].position += wind * dt;
// Each cache line: 64 B / 32 B = 2 entities; the loop
// reads 2 x 12 B = 24 of the 64 B (37.5%).

Structure-of-Arrays (SoA)

struct Entities {
  vec3* positions;
  vec3* velocities;
  float* healths;
  uint32_t* flags;
  int count;
};

// Loop touches positions only:
for (int i = 0; i < entities.count; ++i)
  entities.positions[i] += wind * dt;
// Each cache line: 64 B / 12 B ≈ 5.3 positions, all
// of it read: ~2.7x the useful bytes per miss.

The widget below lays out the same entities both ways. Pick a workload (position only, position plus velocity, or every field): bright fields are the ones the loop reads, thin lines mark 64-byte cache lines, and the stats count the lines each layout loads and the fraction of their bytes the loop uses. AoS wins when the workload touches every field of every entity (because the entity is already on the line); SoA wins when the workload touches one or two fields, which is the common case in physics, rendering, and AI passes:
Live · AoS vs SoA cache trace
touch position
position + velocity
all fields
entity count 128
RUN ▶
RESET ⟲
AoS lines loaded
64
SoA lines loaded
24
AoS useful %
38%
SoA useful %
100%
The top strip is AoS and the bottom SoA; bright fields are the ones the loop reads, and thin lines mark 64-byte cache lines. At 128 entities, press RUN for each workload: touch position gives 64 AoS lines at 38% useful against 24 SoA lines; position + velocity gives 64 against 48; all fields gives 64 against 64. "Useful %" is the fraction of each loaded cache line that the loop actually reads. Touch one fat field (12-byte position) in AoS and you waste about 62% of every line; touch a 4-byte field (health, flags) and you waste about 88%. SoA waste is ~0% on a contiguous field, regardless of which field. When the loop touches every field, AoS catches up because the entity is already on the line. Pick the layout for the dominant traversal, not the rare one. Stoyan Nikolov's CppCon 2018 refactor of an HTML rendering engine shows the same shape on a non-game workload[37].
When AoS is right anyway

SoA isn't always the answer. If you iterate every field of every entity together (a serializer, a debug visualizer that draws all state, an entity that's used as a single unit), AoS is fine and the indirection of SoA's separate arrays costs you. Acton's CppCon 2014 talk puts the rule as solving for the most common case first[38]: find the dominant access pattern, lay out the data for that pattern, and accept that the minor patterns will pay for it. Most engine systems have one dominant pass (physics integrates position and velocity, the renderer iterates transforms, the audio system iterates AudioSources); design for it, not for hypothetical generality.
SoA, archetypes, and ECS

An ECS "archetype" is exactly SoA at the storage level: an archetype is a unique combination of component types, and all entities with that combination live in one chunk where each component type is a contiguous array. Unity's ECS chunks (§14) and Bevy's default table storage work this way. EnTT is deliberately not archetype-based: its model is sparse-set based (a dense array of components plus a sparse index from entity to position)[3], one packed array per component type across all entities, which reaches the same contiguous-per-component layout by a different route. The performance argument is the one in the widget above: a physics system that integrates position and velocity touches only those component arrays, with full cache-line utilization.
Check Your Understanding Q1 of 1

A physics pass integrates 4,000 rigid bodies’ position and velocity (24 of each body’s 96 bytes) every frame. Other systems (rendering, AI) read different fields. Which layout does the dominant pass argue for, and what’s the next decision after layout?
AoS, because position and velocity land on the same cache line as the rest of each body’s state, so one load feeds the whole update.
SoA on position and velocity at minimum; then decide whether the remaining fields get their own arrays or stay packed in a cold struct.
AoS with a virtual update() per body, so each body type can integrate its own state in one place.
0/1 correct

[Back to Memory Allocators from Scratch](index.md)
