# The object-graph problem

A guard is chasing the player when the game saves, and after a reload it should still be chasing the player. Game objects point at each other: an AI's target, a projectile's owner, a node's parent. Those references are pointers in memory, and a pointer is just the address where an object happens to live this run: "pointer values usually change from run to run"[2]. Write a raw pointer to disk and on the next load it points at garbage. The fix is to serialize a reference as something position-independent and re-resolve it on load.

Three ways to encode a reference, from worst to best for a save[2]:

    Raw pointer. Unusable directly: it needs a fix-up table converting saved offsets back to addresses after load, which is what this section avoids.
    Array index. Safe to save, but breaks when items move or are removed, and forces one big homogeneous array.
    Generational handle. A small value, typically {index, generation}, that uniquely names an object and stays valid across a save/load. This is the encoding to prefer.

Fix-up is two passes, and the generation counter is load-bearing

Loading an object graph is two passes[3]: pass 1 instantiates every object so all targets exist, recording id → new-object; pass 2 walks the saved references and re-resolves each id to the freshly-created object. This re-resolution is called pointer fix-up (or swizzling). The generation counter in a handle is what makes a stale reference safe: if the slot was freed and reused, the generations no longer match and the handle resolves to invalid, not to whatever now occupies the slot. This is the same handle from the Gameplay Layer, now crossing a save boundary. Note the two distinct failure modes: a missing target (the referenced object was not saved) needs a null-and-skip; a reused slot is what the generation check catches. They are not the same bug.

Worked example

In the widget below, the Player lives at 0x4a10 this run, and the Guard's target holds 0x4a10. Save that as id #1 instead. On reload the Player comes back at 0x9c80. Pass 1 creates all three objects and records #1 → 0x9c80; pass 2 rewrites the Guard's target and the Turret's owner from #1 to 0x9c80. A saved raw 0x4a10 would point at whatever happens to live there now. For the generation check: a handle saved as {index 7, generation 3} whose slot was freed and reused now finds generation 4 there, so it resolves to invalid instead of the new occupant.

Save the graph, quit, reload. With ids, the references re-resolve through the handle table and reconnect. Flip on "store raw pointers" and the saved addresses point at nothing after reload: the dangling reference this section exists to prevent.
1 · Save
2 · Reload
3 · Fix up
Reset
store raw pointers (the bug)
Press 1, 2 and 3 in order, then Reset, tick store raw pointers and do it again. Save flattens each reference into the blob. Reload brings the objects back at new addresses, so the references are momentarily unresolved (red). Fix up re-resolves them. With ids it reconnects cleanly through the id → object map; flip on raw pointers and fix-up has only the old, now-meaningless addresses, so the references dangle. That is why you serialize ids, never pointers.
Serialize a reference as a handle, re-resolve it on load
C++
Rust

struct Handle { uint32_t index; uint32_t generation; };   // position-independent reference

// SAVE: write the {index, generation}, never the raw pointer.
void writeRef(Writer& out, Handle handle) { out.u32(handle.index); out.u32(handle.generation); }

// LOAD is TWO passes:
//   pass 1: instantiate every saved object, recording id -> new slot (so all targets exist)
//   pass 2: re-resolve each saved handle through the table (the fix-up)
Entity* resolveRef(const HandleTable& table, Handle handle) {
    if (handle.index >= table.slots.size()) return nullptr;   // damaged or foreign save: never index out of range
    const Slot& slot = table.slots[handle.index];
    if (slot.generation != handle.generation) return nullptr;   // stale: slot reused -> invalid, NOT the wrong object
    return slot.entity;                                  // the live pointer, valid this run
}

Serialization libraries enforce this rule for you. cereal refuses raw pointers outright and dedups shared references: data behind a shared_ptr "is serialized only once, even if several shared_ptr... that point to the same data are serialized," and is re-linked on load[4]. Unity's default inline serialization is the cautionary tale: it can't represent null, it turns one object referenced from two fields into two separate copies, and it can't represent cycles[5]. Unity's manual shows the workaround: an ISerializationCallbackReceiver that flattens a tree into an array of nodes, each storing its child count and the index of its first child, the id-reference pattern applied by hand. (Fields marked [SerializeReference] are the exception: they store null, shared and cyclic references.)
