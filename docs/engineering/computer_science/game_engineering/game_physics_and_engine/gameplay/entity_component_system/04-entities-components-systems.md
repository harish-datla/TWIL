# Entities, Components, Systems

A designer wants a homing mine: something that sits still, then moves, then explodes, without a new class. An ECS has three concepts and zero inheritance.

Entity: a generational index. Typically a 32-bit or 64-bit integer split into an index (slot in an array) and a generation counter. The entity itself stores nothing. It is a key into the component tables.

Component: a plain data struct. No methods, no vtable, no inheritance. struct Position { float x, y, z; }; is a complete component. Components are stored in typed, contiguous arrays. The storage strategy (how those arrays are organized) is the subject of sections 5 and 6.

System: a function (or callable) that queries a set of component types and iterates all entities matching that query. A movement system declares "give me every entity with Position and Velocity" and runs position += velocity * dt for each one. Systems have no per-entity state. They read and write components; the ECS runtime provides the iterator.

Worked example

The listing below puts the player at the origin with a Velocity of 1 along x, and the prop at x = 5 with no Velocity. One 1/60 s tick of the movement system moves the player to x = 1 × 1/60 = 0.0167 and never touches the prop, because the prop doesn't match the query. Making the prop move later takes one add<Velocity> and no new class.

This separation has three consequences that matter for performance:

    Homogeneous arrays. All Position components are stored in one flat array. Iteration is a sequential scan. The CPU prefetcher sees a constant stride and loads ahead.
    No virtual dispatch. A system is a single function pointer. It runs in a tight loop over flat data. No indirect call per entity.
    Declarative access. Each system declares which component types it reads and which it writes. The scheduler can run two systems in parallel if their access sets don't conflict. This is mechanical, not hand-tuned.

// Minimal ECS usage pattern (pseudocode).
// Create entities
auto player = world.spawn();
world.add<Position>(player, {0, 0, 0});
world.add<Velocity>(player, {1, 0, 0});
world.add<Health>(player, {100, 100});

auto prop = world.spawn();
world.add<Position>(prop, {5, 0, 0});
world.add<StaticTag>(prop, {});

// Movement system: iterates entities with Position AND Velocity.
// The prop (no Velocity) is excluded automatically.
world.system<Position, const Velocity>(
    [](auto& position, const auto& velocity) {
        position.x += velocity.dx * dt;
        position.y += velocity.dy * dt;
        position.z += velocity.dz * dt;
    }
);

