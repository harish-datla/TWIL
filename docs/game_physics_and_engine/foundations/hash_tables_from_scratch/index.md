# Hash Tables from Scratch

An engine looks things up by key all frame: the texture for an asset path, the compiled shader for a set of feature flags, the session for a player ID. A hash table answers each of those in one or two memory reads instead of a search, as long as it's built well. This page builds one from scratch and shows what makes it fast or slow: the hash function, what happens when two keys want the same slot, how full the table gets, how deletion works and how it resizes, with live demos of each and a look at what shipping engines use.
Time
~60 min
Level
Junior to mid; review for senior
Prereqs
You can read C++ or pseudocode. Arrays, pointers, modular arithmetic. Big O (read the prior tutorial if it feels rusty).
Hardware
None. A feel for cache lines helps in §11.
◂ Build a Game Engine
Phase 0 · Foundations
Next · Sorting Algorithms ▸
What you'll be able to do by the end

    Explain how a hash table turns a key into a slot, and pick a hash function for integer keys, short strings and untrusted input.
    Predict how a table's lookup cost grows as it fills, for chaining, linear probing and Swiss tables, and choose a resize threshold.
    Delete from an open-addressing table without breaking lookups, and spot tombstone buildup in a profile.
    Avoid the frame spike a full rehash causes, with incremental rehashing.
    Recognize the designs in Unreal's TMap, EASTL, ECS entity lookup and Abseil's flat_hash_map, and know when direct indexing beats hashing.


## Sections

- [Why Hash Tables](01-why-hash-tables.md)
- [A Short History](02-a-short-history.md)
- [Hash Functions](03-hash-functions.md)
- [Chaining](04-chaining.md)
- [Open Addressing: Linear Probing](05-open-addressing-linear-probing.md)
- [Quadratic Probing and Double Hashing](06-quadratic-probing-and-double-hashing.md)
- [Robin Hood Hashing](07-robin-hood-hashing.md)
- [Swiss Tables (Flat Hash Maps)](08-swiss-tables-flat-hash-maps.md)
- [Deletion Strategies](09-deletion-strategies.md)
- [Load Factor and Resizing](10-load-factor-and-resizing.md)
- [Cache Behavior and Memory Layout](11-cache-behavior-and-memory-layout.md)
- [Game Engine Case Studies](12-game-engine-case-studies.md)
- [Pitfalls](13-pitfalls.md)
- [What's Next](14-whats-next.md)
