# Lifetime & handles

A rocket kills an enemy in the middle of the update loop, while three other objects still hold references to it. Two lifetime hazards, two standard fixes. Destroying an object mid-update (while the loop iterates the object list) invalidates iterators and leaves dangling references mid-frame, the classic gameplay crash. Deferred destruction is the fix: mark for death now, actually destroy at a safe point (end of frame, before render). Unity does exactly this: Destroy() delays actual destruction until after the Update loop, before rendering[11].
Raw pointers to game objects dangle, use generational handles

Gameplay holds long-lived cross-frame references (an AI's target, a projectile's owner). A raw pointer dangles when the target is destroyed and its slot reused, a silent use-after-free with no way to detect staleness[12]. A generational handle ({index, generation}) bumps the slot's generation on free; a stale handle fails the generation check on resolve and returns null instead of corrupt memory. The handle is never dereferenced blindly; every use goes through that check. The ECS tutorial builds the full allocator; here it's how a gameplay reference survives the target's death.

Worked example

An AI saves a handle {index 2, generation 3} to its target. The target dies, so slot 2's generation becomes 4, and a pickup spawns into slot 2. On the next resolve the handle's 3 doesn't match the slot's 4, so it returns null and the AI picks a new target. A raw pointer to slot 2 would quietly hand back the pickup.

The widget plays that sequence with both kinds of reference:
allocate
free slot 2
resolve refs
A row of slots with their values, generations and alive flags, plus a raw-pointer card and a handle card. Press free slot 2, then allocate, then resolve refs. Both references point at slot 2. Free it (generation bumps), allocate again (the slot is reused by a new object), then resolve: the handle sees the generation mismatch and safely returns null; the raw pointer silently resolves to the new object, a use-after-free reading the wrong entity.

Spawning has the mirror-image problem. An object added to the list during the update loop gets updated later in the same loop, so it acts in the tick it was spawned, before the player has had a chance to see it[7]. Nystrom's simple fix caches the list length when the loop starts, so anything appended after that waits for the next tick.

The program below puts the three rules together: handles for every cross-frame reference, destruction that marks now and frees at the end of the tick, and spawning that waits a tick (a flag holds each new object back until the tick ends, the same trick as the death mark). The update loop indexes the slot array and re-reads it each pass, because a spawn can grow the array and move it in memory, which would leave a held reference or iterator pointing at the old block.
A small object world: handles, deferred destruction, deferred spawning (complete program)
C++
Rust

// A tiny game-object world: generational handles, deferred destruction,
// and objects spawned mid-tick that wait for the next tick to act.
#include <cstdint>
#include <cstdio>
#include <string>
#include <utility>
#include <vector>

struct Handle { std::uint32_t index = 0; std::uint32_t generation = 0; };

enum class Kind { Rocket, Goblin, Archer, Loot };

struct GameObject {
    Kind kind;
    std::string name;
    Handle target;   // a cross-frame reference is a handle, never a pointer
};

class World {
    struct Slot {
        GameObject object{Kind::Loot, "", {}};
        std::uint32_t generation = 0;
        bool alive = false;
        bool dying = false;          // destroyed this tick, freed at the end of it
        bool spawnedThisTick = false;
    };
    std::vector<Slot> slots;
    std::vector<std::uint32_t> freeSlots;

public:
    Handle spawn(GameObject object) {
        std::uint32_t index;
        if (!freeSlots.empty()) { index = freeSlots.back(); freeSlots.pop_back(); }
        else { index = static_cast<std::uint32_t>(slots.size()); slots.emplace_back(); }
        Slot& slot = slots[index];
        slot.object = std::move(object);
        slot.alive = true;
        slot.dying = false;
        slot.spawnedThisTick = true;  // don't let it act before anyone has seen it
        return {index, slot.generation};
    }

    // Null for a handle whose object was freed, or marked for death this tick.
    GameObject* resolve(Handle handle) {
        if (handle.index >= slots.size()) return nullptr;
        Slot& slot = slots[handle.index];
        if (!slot.alive || slot.dying || slot.generation != handle.generation) return nullptr;
        return &slot.object;
    }

    // Mark now, free later: the memory stays valid until endTick().
    void destroy(Handle handle) {
        if (resolve(handle)) slots[handle.index].dying = true;
    }

    template <class UpdateFn>
    void tick(UpdateFn&& updateObject) {
        // Index loop, re-reading slots[i] each pass: spawn() may grow the
        // vector, which would invalidate a reference or iterator held across it.
        for (std::uint32_t i = 0; i < slots.size(); ++i) {
            if (!slots[i].alive || slots[i].dying || slots[i].spawnedThisTick) continue;
            updateObject(*this, Handle{i, slots[i].generation});
        }
        endTick();
    }

private:
    void endTick() {
        for (std::uint32_t i = 0; i < slots.size(); ++i) {
            Slot& slot = slots[i];
            if (slot.dying) {
                slot.alive = false;
                slot.dying = false;
                ++slot.generation;   // every old handle to this slot is now stale
                freeSlots.push_back(i);
            }
            slot.spawnedThisTick = false;
        }
    }
};

void updateObject(World& world, Handle self) {
    GameObject* me = world.resolve(self);
    switch (me->kind) {
    case Kind::Rocket:
        if (GameObject* victim = world.resolve(me->target)) {
            std::printf("  %s hits %s\n", me->name.c_str(), victim->name.c_str());
            world.destroy(me->target);
            world.spawn({Kind::Loot, "loot", {}});   // may reallocate: don't use 'me' after this
            world.destroy(self);
        }
        break;
    case Kind::Archer:
        if (GameObject* target = world.resolve(me->target))
            std::printf("  %s shoots at %s\n", me->name.c_str(), target->name.c_str());
        else
            std::printf("  %s: target is gone, holding fire\n", me->name.c_str());
        break;
    case Kind::Goblin:
        std::printf("  %s growls\n", me->name.c_str());
        break;
    case Kind::Loot:
        std::printf("  %s sparkles\n", me->name.c_str());
        break;
    }
}

int main() {
    World world;
    Handle goblin = world.spawn({Kind::Goblin, "goblin", {}});
    world.spawn({Kind::Rocket, "rocket", goblin});
    world.spawn({Kind::Archer, "archer", goblin});
    world.tick([](World&, Handle) {});   // end the spawn tick so everything is active

    for (int tickNumber = 1; tickNumber <= 2; ++tickNumber) {
        std::printf("tick %d\n", tickNumber);
        world.tick(updateObject);
    }
    // Two newcomers reuse the two freed slots, one of them the goblin's.
    Handle rat = world.spawn({Kind::Goblin, "rat", {}});
    Handle bat = world.spawn({Kind::Goblin, "bat", {}});
    std::printf("rat in slot %u, bat in slot %u\n", rat.index, bat.index);
    std::printf("old goblin handle {%u, %u} resolves to %s\n", goblin.index,
                goblin.generation, world.resolve(goblin) ? "an object" : "null");
}

Both versions print the same trace. On tick 1 the goblin growls, then the rocket hits it, marks the goblin and itself for death, and spawns loot. The archer updates after the rocket in the same tick and already gets null for its target, while the goblin's memory stays valid until the tick ends. The loot first acts on tick 2. Afterwards a rat and a bat take the two freed slots, the bat landing in slot 0 where the goblin lived, and the goblin's old handle {0, 0} still resolves to null because slot 0 is now on generation 1.
What's intentionally missing

    Components. Behavior is a switch on a kind to keep the listing short; a real world dispatches to the components from §3.
    Destruction callbacks. A dying object usually tells others (drop loot, unsubscribe its listeners to avoid §4's lapsed-listener leak) from a hook that runs at the free.
    Generation wraparound. A 32-bit generation wraps after 232 frees of one slot, after which a very old handle could match again. Some allocators retire a slot instead of wrapping.
    Dense iteration. Dead slots stay in the array and the loop skips them. An ECS packs live components together instead (the ECS tutorial).
    Threads. Everything runs on one thread. A parallel update needs the spawn and destroy lists to be per-thread or locked.

