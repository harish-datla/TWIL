# The Pool Allocator (Fixed-Size Free List)


Bullets spawn and die hundreds of times a second, in no particular order, and every one is 64 bytes. A pool allocator is built for that: one size of slot, many slots, a free list threaded through the empty ones. Allocate pops the head of the free list. Free pushes the slot back onto the head. Both operations are O(1) and the slots are interchangeable, which gives the allocator a property linear allocators don't have: per-object free in any order, with no external fragmentation, because every hole is exactly the same shape as every fit. Wilson's survey calls this "simple segregated storage"[1]; Alexandrescu's Loki SmallObjectAllocator is the C++ template version[15]; Boost.Pool is a widely used open-source one[29].

The widget shows why a pool can't fragment. Allocate a few slots, then free one at random. The hole is a single slot, the same size as every other slot, and it becomes the new free-list head, so the next alloc reuses it without searching. There is no "I have 96 bytes free but no contiguous block big enough" failure mode here, because every block is already the right size:
Live · Pool of 24 fixed-size slots
ALLOC ◯
FREE RANDOM ✕
FILL ALL
RESET ⟲
allocated
0 / 24
free list head
0
free list length
24
external frag
0%
Each cell is a slot: colored ones are allocated, dim ones are free and labeled with the next free index, and the yellow box marks the free-list head. Press ALLOC three times, FREE RANDOM once, then ALLOC again: the slot you just freed comes back first, with no search. The free list (dashed links) is stored in the slot bytes themselves, costing zero per-slot overhead. Each free slot's first sizeof(int) bytes hold the index of the next free slot. The "external fragmentation" stat is always zero by construction: every free slot is a fit for every allocation, because every allocation is the same size.
The free list lives in the empty slots
pool_allocator.cpp · O(1) alloc and free, zero overhead per slot

template <typename T, size_t kSlotCount>
class PoolAllocator {
  static_assert(sizeof(T) >= sizeof(int),
                "slot must be big enough to hold a free-list index");
  static_assert(kSlotCount >= 1 && kSlotCount <= INT_MAX,
                "slot indices must fit in an int");

  // Storage: kSlotCount slots, each sized like T and aligned for T. unsigned char,
  // because only unsigned char and std::byte arrays formally provide storage for objects.
  alignas(T) unsigned char storage[kSlotCount * sizeof(T)];

  // Free-list head: index of the next free slot, or -1 if pool is full.
  int freeHead = 0;

  unsigned char* slotPtr(int slot) { return storage + size_t(slot) * sizeof(T); }

  // The link lives in a free slot's first bytes. memcpy, because no int object exists
  // there and T may be less aligned than int; compilers emit a single load or store.
  int  readLink(int slot)            { int next; std::memcpy(&next, slotPtr(slot), sizeof next); return next; }
  void writeLink(int slot, int next) { std::memcpy(slotPtr(slot), &next, sizeof next); }

public:
  PoolAllocator() {
    // Thread the free list through every slot: slot[i] -> slot[i+1], last -> -1.
    for (int i = 0; i < int(kSlotCount) - 1; ++i) writeLink(i, i + 1);
    writeLink(int(kSlotCount) - 1, -1);
  }

  // O(1): pop the free-list head. Returns raw storage; construct with placement new.
  T* allocate() {
    if (freeHead < 0) return nullptr;        // free list empty: pool exhausted
    int slot = freeHead;
    freeHead = readLink(slot);                 // the popped slot's link is the new head
    return reinterpret_cast<T*>(slotPtr(slot));
  }

  // O(1): push the slot back onto the free-list head. Destroy the object first.
  void deallocate(T* p) {
    int slot = int((reinterpret_cast<unsigned char*>(p) - storage) / sizeof(T));
    writeLink(slot, freeHead);                 // next = old head
    freeHead = slot;                           // new head
  }
};

Worked example

A fresh pool of 24 slots: the free list runs 0 → 1 → 2 → … → 23, and the head is 0.

Three allocations pop slots 0, 1 and 2. The head is now 3.

Free slot 1: write the old head, 3, into slot 1's first bytes, and set the head to 1. The list now runs 1 → 3 → 4 → …

The next allocation pops slot 1 and reads its link, so the head is back to 3. Each step is one read and one write, whatever order the frees came in.

The free list costs no extra memory: each empty slot's first few bytes hold the link, since nothing else lives there. A pool of 24 slots of 96-byte particles uses exactly 24 * 96 = 2304 bytes plus a single 4-byte head index. A general-purpose allocator typically adds 8 to 16 bytes of header per allocation, so for small objects the pool also packs more of them into each cache line.
The internal-fragmentation tax

Pools are immune to external fragmentation (free space between allocations) but pay internal fragmentation (slot bigger than what the caller actually needed). A pool of 96-byte slots holding 80-byte payloads wastes 16 bytes per slot, forever. The fix is to have several pools, one per common size, and pick the smallest pool whose slot fits. That's what the slab allocator does at scale, and it's what most production small-object allocators do, including Loki's[15].
Check Your Understanding Q1 of 2

A pool of 1,024 slots of Bullet (sizeof = 64 bytes). After spawning 800 bullets and freeing every third one, the pool reports 533 allocated slots and a free list of length 491. Are the free slots scattered or contiguous, and does it matter for the next allocate()?
Scattered, and it does not matter for the next allocation.
Contiguous, because the pool defragments after each free.
Scattered, and the next allocation will fail unless the heap is compacted.
0/2 correct

[Back to Memory Allocators from Scratch](index.md)
