# Case studies

Six shipping designs show the trade-offs above made in different directions.
Overwatch (Blizzard, 2016)

Timothy Ford's GDC 2017 talk[7] describes the custom ECS behind Overwatch. The talk itself sits behind the GDC Vault paywall; its public abstract says the ECS is used "to create a rich variety of gameplay" and "to curtail complexity" as heroes are added, and that the game's networked simulation "leverages determinism to achieve responsiveness and precision." That pairing is the general appeal of ECS for netcode: when systems run in a fixed order over plain data, the same inputs produce the same state, which is what client prediction and rollback need.
Unity DOTS

Unity's archetype ECS stores entities in 16 KiB chunks[8]. Each chunk belongs to one archetype. Within a chunk, component arrays are laid out in SoA order at the component-type level: all Position structs contiguous, then all Velocity structs, then all Health structs. The Burst compiler compiles jobs written in a restricted subset of C# to native code through LLVM and auto-vectorizes loops over these arrays where it can, emitting SIMD instructions without manual intrinsics. The C# Job System runs those jobs on worker threads, with dependencies derived from each system's declared component access[10], similar in spirit to Bevy's executor.
Bevy (Rust)

Bevy[18] uses archetype storage with a multi-threaded system executor. Systems are plain Rust functions. Their parameter types encode the query: Query<(&Position, &mut Velocity)> requests read access to Position and write access to Velocity for every entity that has both. The data the system wants goes in the first type parameter as a tuple; the optional second parameter is a filter, not more data, which is where With<Player> or Changed<Health> go. The executor builds a dependency graph from these signatures and dispatches non-conflicting systems to a thread pool. Storage is per-component rather than fixed: a component defaults to Table (columnar, "fast and cache-friendly iteration, but slower addition and removal") and can opt into SparseSet ("fast addition and removal of components, but slower iteration")[19]. Bevy is the clearest counterexample to treating archetype and sparse set as a whole-library choice. The engine keeps moving fast enough that version matters when you read its docs: 0.19 unified resources as singleton entity components, replaced the render graph with ordinary ECS schedules, and exposed contiguous query access as table slices so system loops can be vectorized[22].
Flecs (C/C++)

Flecs[15] is an archetype-based ECS with first-class relationships. The (ChildOf, parent) relationship pair acts as a component: entities with the same parent share an archetype, making "find all children of X" an archetype-table scan. Flecs caches query results and maintains an archetype graph where edges represent "add component C" or "remove component C" transitions, so a structural change finds its destination archetype by following a cached edge instead of hashing the new component set.
EnTT (C++)

EnTT[12] is the primary example of a sparse-set ECS. Each component type has its own sparse set pool. No archetype tables, no table migration on add/remove. The trade-off: iteration over multiple component types requires intersecting multiple sparse sets (iterating the smallest set and looking up each entity in the others). Used in Minecraft Bedrock Edition. Michele Caini's "ECS back and forth" blog series documents the design decisions, including the owning groups from §5[14].
Unreal Mass Entity

Mass Entity[23] is Epic's archetype-based framework in Unreal Engine 5, used for crowd and traffic simulation in the City Sample and The Matrix Awakens[24]. Epic's own documentation states the model without the numbers: entities of identical composition share an archetype, and "Entities in an Archetype are organized in memory Chunks"[23]. It publishes no chunk size. The widely repeated "128 bytes per cache line, 1024 cache lines" figure comes from the community MassSample documentation, whose authors state they are not affiliated with Epic and describe it as their reading of the UE::Mass::ChunkSize constant[25]. Read literally it is a 128 KB chunk, an order of magnitude larger than Unity's 16 KiB. Treat the derivation as unofficial. Mass interoperates with Unreal's Actor/Component model through traits that bridge the ECS world and traditional UObjects.
Check Your Understanding Q1 of 2

Your game has a particle system that spawns 10,000 particles per frame, each living for 0.5 seconds. Particles have Position, Velocity, Lifetime, and a Color component. Components are added at spawn and never modified (no add/remove after creation). Which ECS storage model has the least overhead for this workload?
Archetypes, since every particle shares one component set and iteration stays sequential.
Sparse sets, since particles are spawned and destroyed constantly every frame.
Neither, because the spawn rate dominates whatever storage layout you pick.
0/2 correct
