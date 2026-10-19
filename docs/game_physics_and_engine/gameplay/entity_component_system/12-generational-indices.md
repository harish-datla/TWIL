# Generational indices

A missile stores the ID of the ship it's tracking; the ship explodes, and a new pickup spawns into the same slot a frame later. Entity IDs must be recycled. A game that spawns and destroys thousands of projectiles per second will grow the slot array unboundedly unless IDs are reused, since per-ID storage (sparse arrays, component slots) scales with the high-water mark of the index. A monotonically increasing 32-bit index would also run out: at 10,000 spawns a second it wraps in about five days. The standard solution: a generational index.

The entity allocator maintains an array of slots. Each slot has a generation counter and an alive flag. A free list tracks available slots. Allocating pops a slot off the free list and returns {index, generation}. Freeing increments the generation and pushes the slot back onto the free list. Any saved reference that holds the old generation will fail the generation check on resolve. Weissflog states the rule in one line: "Each array slot gets its own generation counter, which is bumped when a handle is released"[28], and Gray states the matching test on the read side: "To test if a weak pointer is still valid, we check whether the generation in the weak pointer's struct matches the generation in the slot indicated by the id"[29].

This prevents the ABA problem in entity references: slot 5 held enemy A (generation 0), enemy A was destroyed (generation bumped to 1), slot 5 was reused for projectile B (generation 1). A stale reference to {index: 5, generation: 0} correctly fails to resolve, even though slot 5 is alive again.

Worked example

Pack a 32-bit handle as a 20-bit index and a 12-bit generation and you get 220 = 1,048,576 live slots, each reusable 4,096 times before its generation wraps around and an ancient stale handle could match again. A plain 32-bit counter with no reuse would wrap in 232 ÷ 10,000 ≈ 430,000 s, about five days, at 10,000 spawns a second.
Live · Generational index allocator
allocate
free random
resolve random ref
reset
Allocator reset. 12 slots available.
Allocated e[0] gen=0 (saved as ref_0g0)
Allocated e[1] gen=0 (saved as ref_1g0)
Allocated e[2] gen=0 (saved as ref_2g0)
Allocated e[3] gen=0 (saved as ref_3g0)
alive
4
free slots
8
saved refs
4
Slots with their generations, a list of saved references, and a log. Allocate, free and resolve a few times, and resolve a reference after its slot was freed and reused. Each allocation saves a reference (index + generation). Freeing a slot bumps its generation. Resolving a saved reference checks the generation against the slot's current generation. If they disagree, the reference is stale: the entity it pointed to was destroyed, and the slot may have been reused for a different entity. This is how ECS implementations detect use-after-free without garbage collection.
Generational index allocator
C++
Rust

struct Entity {
    uint32_t index;
    uint32_t generation;
};

struct EntityAllocator {
    struct Slot { uint32_t generation; bool alive; };

    std::vector<Slot> slots;
    std::vector<uint32_t> freeList;

    Entity allocate() {
        uint32_t index;
        if (!freeList.empty()) {
            index = freeList.back();
            freeList.pop_back();
        } else {
            index = static_cast<uint32_t>(slots.size());
            slots.push_back({0, false});
        }
        slots[index].alive = true;
        return { index, slots[index].generation };
    }

    void free(Entity entity) {
        if (!isAlive(entity)) return;
        slots[entity.index].alive = false;
        slots[entity.index].generation++;           // invalidate stale refs
        freeList.push_back(entity.index);
    }

    bool isAlive(Entity entity) const {
        return entity.index < slots.size()
            && slots[entity.index].alive
            && slots[entity.index].generation == entity.generation;
    }
};

What's intentionally missing

The allocator above is the minimal viable version. Production implementations add: packing index and generation into a single 64-bit integer (Bevy uses 32 bits index + 32 bits generation), tombstone detection for double-free, configurable generation width (some engines use 16-bit generation + 16-bit index for tighter packing), and atomic operations for thread-safe allocation.
