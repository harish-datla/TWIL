# Queries and query caching

A world with two hundred archetypes and fifty systems can't afford to work out from scratch, every frame, which tables each system should walk. A query is the interface between a system and the storage. A query descriptor says: "give me every entity that has all of these component types and none of those." The ECS runtime resolves this into a set of archetype tables whose component sets satisfy the constraints.

The resolution step is a bitset intersection. Assign each component type a bit index. Each archetype stores a bitmask of its component types. The query "With(Position, Velocity), Without(Static)" becomes:

// Query: With(Position, Velocity), Without(Static)
// The 'ull' suffix matters: a bare 1 is an int, so 1 << 32 and above is
// undefined behavior (and 1 << 31 was too, before C++20). Declaring the result
// uint64_t does not help, because the shift has already happened in int.
uint64_t withMask    = (1ull << POSITION_BIT) | (1ull << VELOCITY_BIT);
uint64_t withoutMask = (1ull << STATIC_BIT);

for (auto& archetype : allArchetypes) {
    bool hasAll    = (archetype.mask & withMask) == withMask;
    bool hasNone   = (archetype.mask & withoutMask) == 0;
    if (hasAll && hasNone) {
        iterateArchetype(archetype);           // sequential scan of matching table
    }
}

With Position as bit 0, Velocity bit 1, Health bit 2 and Static bit 3, the query's masks are 0b0011 (with) and 0b1000 (without). The archetype {Position, Velocity, Health} has mask 0b0111: AND with 0b0011 gives 0b0011, all required bits present, and AND with 0b1000 gives 0, so it matches. The archetype {Position, Velocity, Static}, 0b1011, fails the second test.

This outer loop (over archetypes) is cheap: archetypes number far fewer than entities, and each is checked by two bitwise ANDs and two comparisons. The inner loop (over entities in each matching archetype) is the sequential scan that does the real work.

Query caching avoids re-running the archetype match every frame. On first execution, the query finds all matching archetypes and stores pointers to them. When a new archetype is created (because some entity got a novel component combination), it is tested against the cached queries and added where it matches, either immediately when the archetype is created (Flecs[15]) or the next time the query runs (Bevy's QueryState[18]).
Live · Query matcher
Click components to cycle: off / with / without
Pos: withVel: withHP: offSprite: offAI: offStatic: without
archetypes matched
4/7
entities matched
2,600
Each row is an archetype with its component bits; matched ones light up and the entity count updates. Toggle components between with, without and ignored. The WITH bitmask requires all marked bits to be set in the archetype. The WITHOUT bitmask requires all marked bits to be clear. Two bitwise ANDs per archetype. The entity count shows how many entities the system would iterate after matching. Toggle the component constraints to see archetypes drop in and out of the match set.
