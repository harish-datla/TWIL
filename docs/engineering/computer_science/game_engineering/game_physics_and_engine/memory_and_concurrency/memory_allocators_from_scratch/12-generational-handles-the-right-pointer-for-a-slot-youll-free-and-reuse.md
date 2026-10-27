# Generational Handles: The Right Pointer for a Slot You'll Free and Reuse


A homing missile stores a pointer to its target. The target dies, its memory is reused for a health pickup, and the missile turns to chase the pickup. A raw pointer is what the compiler hands you: a 64-bit address into the address space, valid until something frees it. The "until something frees it" is the trouble. In a game where entities spawn and die every frame, holding a raw pointer to an entity is a contract you can't enforce: by the time you dereference it, the slot may belong to a different entity. The C++ answer is shared_ptr and weak_ptr, which charge an atomic refcount and a control block per object. The engine answer is the generational handle: 32 or 64 bits split into an index (which slot) and a generation (which generation of that slot). Freeing bumps the generation. A handle is valid if and only if handle.gen == slot.gen. A stale handle fails the check instead of reaching the wrong object; a live one costs a single cache-line load to validate. Frykholm's 2011 BitSquid post is the standard engine-side description[23]; floooh's "Handles are the better pointers" makes the case for them in C APIs[39]; Bevy's Entity[2] and EnTT's entt::entity[3] are widely used ECS implementations.

The widget shows the lifecycle. Spawn creates an entity, Store handle keeps a copy of the newest entity's handle, and Free latest frees the newest live entity and bumps its slot's generation. A handle stored before the free still points to the same slot index but has the old generation, and the lookup correctly rejects it as stale:
Live · Generational handles (8 slots)
SPAWN ✦
FREE LATEST ✕
STORE HANDLE 📌
VALIDATE ALL ?
RESET ⟲
live entities
0 / 8
stored handles
0
live handles
0
stale handles
0
Each box is a slot with its current generation; stored handles are listed below as index : generation, crossed out once stale. Press SPAWN twice, STORE HANDLE, FREE LATEST, SPAWN, STORE HANDLE, then VALIDATE ALL: both stored handles name slot 1, and one is live while the other is stale. A handle is a value, not a pointer. It stays safe to check after its entity dies (the slot index never moves) and reports that death (the slot's generation moved on). A 32-bit handle is half the size of a 64-bit pointer and carries its own validity check.
The 32-bit handle
handle.cpp · 24-bit index, 8-bit generation

struct Handle {
  uint32_t bits;                       // 24 bits index | 8 bits generation

  static Handle make(uint32_t index, uint8_t generation) {
    return { (index & 0xFFFFFF) | (uint32_t(generation) << 24) };
  }
  uint32_t index()      const { return bits & 0xFFFFFF; }
  uint8_t  generation() const { return uint8_t(bits >> 24); }
};

template <typename T>
class HandleTable {
  struct Slot {
    T       value;
    uint8_t generation;                // same width as the handle's field, so the two
    bool    alive;                     // wrap together (see the note on wrap below)
  };
  std::vector<Slot> slots;
  std::vector<uint32_t> freeIndices;  // indices ready for reuse

public:
  Handle spawn(const T& value) {
    uint32_t slotIndex;
    if (!freeIndices.empty()) {
      slotIndex = freeIndices.back(); freeIndices.pop_back();
    } else {
      assert(slots.size() < (1u << 24));  // the index field is 24 bits wide
      slotIndex = uint32_t(slots.size());
      slots.push_back({});
    }
    slots[slotIndex].value = value;
    slots[slotIndex].alive = true;
    return Handle::make(slotIndex, slots[slotIndex].generation);
  }

  void free(Handle handle) {
    if (!isValid(handle)) return;
    Slot& slot = slots[handle.index()];
    slot.alive = false;
    slot.generation++;                      // retires every prior handle to this slot
    freeIndices.push_back(handle.index());  // slot can now be reused
  }

  bool isValid(Handle handle) const {
    const uint32_t slotIndex = handle.index();
    return slotIndex < slots.size()
        && slots[slotIndex].alive
        && slots[slotIndex].generation == handle.generation();
  }

  T* get(Handle handle) {
    return isValid(handle) ? &slots[handle.index()].value : nullptr;
  }
};

Worked example

An entity lives in slot 5, whose generation is 3. Its handle packs both: 5 | (3 << 24) = 0x03000005.

The entity dies. free bumps slot 5's generation to 4 and puts the slot back on the free list. A new entity spawns into slot 5 and gets handle 0x04000005.

The missile still holds 0x03000005. isValid reads index 5, finds generation 4, compares it with the handle's 3, and returns false: the lookup fails instead of handing back the new entity.

24 bits of index supports 16,777,216 slots, far more than most games keep alive at once. 8 bits of generation supports 256 distinct generations per slot before it wraps. Wrap is the only failure mode: after 256 free-and-reuse cycles on the same slot, a handle from generation N matches generation N+256 and validates incorrectly. The mitigations are a wider generation field, or retiring a slot for good instead of reusing it once its generation is about to wrap. EnTT uses 12 bits of generation by default[3]; Bevy uses 32 bits[2].
Why this is what shared_ptr can't be

A shared_ptr costs a control block per object holding atomic strong and weak counts (allocated separately unless make_shared fuses it with the object; 24 bytes in libstdc++ for a plain pointer with the default deleter), and every copy does an atomic increment on it. Detecting death takes a weak_ptr, and its lock() is another atomic operation. A generational handle costs one 32-bit word per handle and one generation field per slot; validation is a bounds check and a compare, and copying a handle copies an integer. The table itself still needs synchronization if one thread frees slots while another looks them up, but that is one lock or one ownership rule per table, not an atomic per copy. P0661 proposed standardizing the pattern as std::slot_map[40]; the C++ standards committee (WG21) didn't adopt it, and engines keep shipping their own.
Check Your Understanding Q1 of 2

An enemy fires a projectile that targets a particular zombie via a handle. Between fire and impact, the zombie dies. The projectile dereferences its handle and gets...
A pointer to whatever entity is now in that slot, which could be a powerup, an audio source, or a different enemy.
A null pointer (or equivalent invalid result): the slot’s generation was bumped on free, so isValid(handle) fails and the lookup refuses the stale handle.
A use-after-free: the slot’s memory went back to the allocator when the zombie died, so the dereference reads freed memory.
0/2 correct

[Back to Memory Allocators from Scratch](index.md)
