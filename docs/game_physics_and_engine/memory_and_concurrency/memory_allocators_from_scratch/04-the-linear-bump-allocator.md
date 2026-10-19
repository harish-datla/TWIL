# The Linear (Bump) Allocator


Every frame the renderer builds thousands of small temporary arrays (visible-object lists, sort keys, command packets), and every one of them is garbage by the next frame. Freeing them one at a time is wasted work, because they all die at the same moment. The linear allocator is built for exactly that, and it's the simplest allocator that's still useful. One pointer. Every allocation moves the pointer forward by the requested size. There is no per-object free. The entire region resets in one instruction at the end of the frame, the level, or whatever lifetime the caller picked. Unreal calls its version FMemStack[5]; Naughty Dog's engine runs per-thread linear allocators on 2 MiB blocks from its tagged heap[24]; Apache calls a sibling design apr_pool. Hanson 1990 is the usual academic reference[13]; Ryan Fleury's "Untangling Lifetimes" essay is a thorough modern walkthrough[26].

The widget is the picture to keep in your head. Click alloc 32 to grab 32 bytes; the head pointer moves. Free is the reset button: one assignment, all 256 bytes released at once. Try to allocate past the end and you get the only failure mode this allocator has, out-of-memory:
Live · Bump-pointer allocator
ALLOC 16
ALLOC 32
ALLOC 64
ALLOC 128
RESET ⟲
head pointer
0
used
0 / 256
allocations
0
overhead
24 B
The strip is the 256-byte region, colored spans are allocations, and the marker is the head pointer. Press ALLOC 64 three times, then ALLOC 128: the head stops at 192 and the fourth call fails with 64 B left. RESET frees everything in one store. Each allocation is a pointer increment. The "overhead" is the allocator state itself: the three pointers (base, head, end) the code below keeps, 24 bytes on a 64-bit build. There is no per-allocation header, because there is no per-allocation free. The failure mode of misuse is just as simple: reset releases every allocation at once, so a caller that keeps a pointer past the reset is holding memory the next frame will overwrite.
The code, in about 30 lines
linear_allocator.cpp · the entire allocator

// One pointer of state. No headers, no free lists, no locks.
struct LinearAllocator {
  char* base;         // start of the region
  char* head;         // next free byte
  char* end;          // one past the end

  void init(void* memory, size_t bytes) {
    base = head = (char*)memory;
    end  = base + bytes;
  }

  // Allocate `size` bytes aligned to `alignment` (default 16, SIMD-friendly).
  void* allocate(size_t size, size_t alignment = 16) {
    // Round head up to alignment (alignment is required to be a power of two).
    uintptr_t raw     = reinterpret_cast<uintptr_t>(head);
    uintptr_t aligned = (raw + alignment - 1) & ~(uintptr_t(alignment) - 1);
    size_t padding   = aligned - raw;            // bytes skipped to reach alignment
    size_t available = end - head;
    // Compare sizes, not pointers: a pointer computed past `end` is undefined behavior,
    // and checking padding first keeps the subtraction from wrapping.
    if (padding > available || size > available - padding) return nullptr;  // out of memory
    char* result = head + padding;
    head = result + size;
    return result;
  }

  // No per-object free. The whole region resets in one assignment.
  void reset() { head = base; }

  size_t used()     const { return head - base; }
  size_t capacity() const { return end - base; }
};

Every line of the allocator's hot path is on screen. allocate is a load, a round-up, a bounds check, a store. reset is a single store. The only branch is the out-of-memory check, and the alignment math is a power-of-two mask, so a call costs a few nanoseconds.

Worked example

The region starts at address 0x1000. Allocate a 12-byte Vec3, then a 96-byte Particle, both with the default 16-byte alignment.

First call: head is already aligned, so the Vec3 lands at 0x1000 and head moves to 0x100C.

Second call: round up, (0x100C + 15) & ~15 = 0x101B & ~0xF = 0x1010. That skips 4 bytes of padding. The Particle lands at 0x1010 and head moves to 0x1010 + 96 = 0x1070.

112 bytes used for 108 bytes of data. The 4 padding bytes are this allocator's only waste.
What's intentionally missing

This is the teaching version, not a shipping one. A production linear allocator adds: a chain of pages so it can grow when one region fills, an explicit "checkpoint" or "mark" API so the engine can rewind to a saved head (that's the stack allocator in the next section), a debug build that fills the region with a poison byte on reset (MSVC's debug heap uses 0xDD for freed memory) to catch use-after-reset bugs, and a thread-local instance or an atomic head so calls don't need a lock. Frykholm's "Custom Memory Allocation in C++" shows the allocator interface these layers plug into, with frame, pool, heap, and tracking-proxy allocators behind it[27].
Check Your Understanding Q1 of 2

A particle system allocates one Particle (96 bytes) and one Vec3 trail vertex (12 bytes) for each particle, all per-frame. You replace new with a 64 KB per-frame linear allocator. After one frame of 500 particles the head pointer should sit at roughly:
500 * 108 = 54,000 bytes, give or take alignment padding.
108 bytes, because the second allocation overwrites the first.
Cannot be determined without running it: the head position depends on what else the heap served that frame.
0/2 correct

[Back to Memory Allocators from Scratch](index.md)
