# Allocators and Why Games Override Them


A console title has a fixed memory budget per subsystem, and a profile shows thousands of small malloc calls per frame from temporary containers that are all thrown away at the end of the frame. Both problems come down to where containers get their memory. Every standard container except std::array takes an allocator template parameter (the adaptors pass it through to the container they wrap). The default is std::allocator<T>, which forwards to ::operator new and ::operator delete. Those usually sit on the platform's thread-safe general-purpose malloc (glibc's malloc, the Windows CRT heap, or a replacement such as jemalloc or tcmalloc), and that's the right answer for most code.

Game engines commonly override it for three reasons[5]:

    Memory budget. Each subsystem gets a fixed pool. The renderer's containers allocate out of the renderer's pool; the AI's out of the AI's. Out-of-memory is a budget violation that's tractable to debug, not a process-wide crash.
    Frame allocators. A linear arena: one big block plus an offset that only moves forward, reset at the start of each frame. Allocating from it is a pointer bump plus an alignment round-up; freeing is a no-op until the frame ends. Per-frame container allocations cost almost nothing once containers draw from it.
    Alignment. SSE/AVX/NEON vector types want 16- or 32-byte alignment. malloc guarantees 16 bytes on 64-bit glibc, Windows, and macOS (8 bytes on 32-bit Windows), which is not enough for a 32-byte-aligned AVX type like __m256. Since C++17, std::allocator routes over-aligned element types through aligned operator new, so std::vector<__m256> is safe on a conforming toolchain. Engine allocators that forward to plain malloc, and any codebase still on C++14, have to supply the over-alignment themselves before putting __m256 or any alignas(32) struct in a container.

Worked example

A bump arena that rounds every allocation up to 16 bytes, starting at offset 0. Allocate 100 bytes: it gets offset 0, and the next free offset is 100, rounded up to 112. Allocate 24: offset 112, next free 136, rounded to 144. Allocate 64: offset 144, next free 208.

Three allocations, three additions and roundings, no locks and no free lists. Resetting the frame sets the offset back to 0.

C++17 added std::pmr (polymorphic memory resources), which lets you pick an allocator at runtime instead of at type-instantiation time. std::pmr::vector<int> is a vector of int that takes its memory resource as a constructor argument. That lets engine pools plug in without making every container a different type, at the cost of a virtual call per allocation. The standard ships std::pmr::monotonic_buffer_resource (a linear arena) and std::pmr::unsynchronized_pool_resource (size-class pools with no locking, for use from one thread) out of the box.
pmr_frame.cpp · per-frame linear arena, no malloc on the hot path

// One frame's worth of scratch storage. Reused every frame.
// (Past the 4 MB, the arena falls back to its upstream resource, the heap.)
static std::array<std::byte, 4 * 1024 * 1024> frameArenaStorage;
std::pmr::monotonic_buffer_resource frameArena(
    frameArenaStorage.data(), frameArenaStorage.size());

void simulateFrame() {
  frameArena.release();    // "free everything from last frame": rewinds to the start of the buffer

  // The vector allocates from the arena: reserve() is one pointer bump, and
  // any growth past 1024 is another bump (the old block is simply abandoned).
  std::pmr::vector<VisibleEntity> visibleEntities(&frameArena);
  visibleEntities.reserve(1024);

  for (const Entity& entity : worldEntities)
    if (mainCameraFrustum.contains(entity.bounds))
      visibleEntities.push_back({entity.id, entity.transform});

  submitDraws(visibleEntities);
  // visibleEntities goes out of scope here. For trivially destructible
  // element types, no destructors run; the buffer's deallocate() on a
  // monotonic arena is a no-op, so the whole teardown is free until
  // frameArena.release() reclaims the arena at the top of next frame.
}


[Back to C++ Containers](index.md)
