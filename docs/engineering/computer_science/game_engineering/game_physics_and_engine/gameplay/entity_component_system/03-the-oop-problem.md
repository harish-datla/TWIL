# The OOP problem

The classical game-object hierarchy starts reasonable: GameObject at the root, RenderableObject inherits from it, PhysicsObject inherits from it, Character inherits from both. By the time a shipped game has a few hundred object types, the hierarchy can be many levels deep, and the problems are structural.

    Diamond inheritance. A FlyingEnemy needs both Enemy (AI, health) and FlyingObject (flight model). Both inherit from PhysicsObject. C++ virtual inheritance "solves" this at the cost of extra indirection, vtable complexity, and a data layout that no one on the team can draw on a whiteboard.
    Fat base classes. Every feature that "most objects need" migrates upward. GameObject accumulates a transform, a bounding box, a name, a layer mask, a tag, an enable flag, a serialization hook. Objects that need none of these (a trigger zone, a sound emitter) pay for all of them in memory and initialization cost.
    Virtual dispatch overhead. A per-frame Update() call on 50,000 objects through a vtable means 50,000 indirect function calls, each one a chance for the branch predictor to miss the target. Agner Fog measures the misprediction penalty at 15 to 20 cycles on Haswell through the Skylake derivatives, and around 18 on Zen 1 to 3[5]. Size the worst case honestly: 50,000 misses at ~17 cycles is about 0.3 ms at 3 GHz, and that is the ceiling, not the expectation. Indirect branch predictors handle a loop that keeps calling the same concrete type well, so a run of same-type objects costs far less. The dispatch bill only approaches that ceiling when the type actually varies from object to object.
    Cache-hostile layout. Each object is heap-allocated. The new allocator interleaves objects of different types in address space. Iterating all RenderableObject instances pointer-chases through a linked list or flat pointer array, loading one cache line per object. Most of that cache line is wasted on fields the current loop does not touch.

The "everything is a GameObject" model that Unity (pre-DOTS) and many custom engines used is a partial fix. It replaces inheritance with composition at the object level, but the components themselves are still polymorphic, heap-allocated, and pointer-chased. The iteration pattern (for each entity, fetch its component by type, call a virtual method on it) is fundamentally the same pointer chase.
Check Your Understanding Q1 of 2

A codebase has a 10-level deep inheritance hierarchy for game objects. Adding a new "magnetic projectile" requires both the Projectile and MagneticField base classes, which share a common PhysicsObject ancestor. What is the most pragmatic fix?
Use virtual inheritance so the shared PhysicsObject base is stored once.
Use plain multiple inheritance and disambiguate the duplicated base by hand.
Drop the hierarchy and compose the entity from Projectile, Magnetic and Physics.
0/2 correct
