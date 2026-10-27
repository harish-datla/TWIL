# The Stack Allocator (Linear with Markers)


Loading a level needs a few megabytes of scratch for parsing manifests and decompressing assets, and all of it should vanish the moment the load finishes, while whatever the frame allocated before the load stays put. A stack allocator handles that. It's a linear allocator with one extra rule: you can save the head pointer ("push a marker") and rewind to it later ("pop to a marker"). Allocations are still pointer bumps; freeing happens in LIFO (last in, first out) order at the granularity of a marker, not an individual object. This is what the C call stack is, generalized: a function pushes its locals on entry and pops them on exit, and the caller does the same one level up. Engines use the same shape for nested scopes that have their own scratch: load a level, push a marker, allocate everything the loader needs, pop the marker, and every byte the loader touched is free in one instruction.

Try the widget. Allocate a few blocks, hit save marker, allocate a few more, then pop to marker. Everything after the marker is gone in one step; everything before it stays:
Live · Stack allocator with markers
ALLOC 32
ALLOC 64
SAVE MARKER ⌃
POP TO MARKER ⌄
RESET ⟲
head
0
markers
0
allocations
0
used
0 / 384
The strip is the 384-byte region, colored spans are allocations, and vertical lines are saved markers. Try the worked example: ALLOC 32, ALLOC 64, SAVE MARKER, ALLOC 64, ALLOC 32, then POP TO MARKER, and the head drops from 192 back to 96 with 2 allocations released. Markers act like a return address for memory. Pop is one assignment; every byte past the marker is released at once. Out-of-order frees are not supported: pop is always to the most recent marker. If your lifetimes are nested, that's the rule you want; if they aren't, a pool or a free-list is the right choice instead.

Worked example

Allocate 32 bytes, then 64: head = 96. Save a marker, which records 96.

The loader allocates 64 bytes and then 32: head = 192.

Pop to the marker: head = 96 again. The loader's 96 bytes are free in one store, and the first two allocations, below the marker, are untouched. Popping a second time with no marker left would be a bug, and the widget refuses it.
The marker pattern in code
stack_allocator.cpp · markers + RAII scope

struct StackAllocator {
  char* base;
  char* head;
  char* end;

  using Marker = char*;          // just the head value at the time of save

  Marker mark() const { return head; }
  void   pop(Marker m)    { head = m; }   // rewind to the saved head

  // Same bump as LinearAllocator::allocate.
  void* allocate(size_t size, size_t alignment = 16) {
    uintptr_t raw     = reinterpret_cast<uintptr_t>(head);
    uintptr_t aligned = (raw + alignment - 1) & ~(uintptr_t(alignment) - 1);
    size_t padding   = aligned - raw;
    size_t available = end - head;
    if (padding > available || size > available - padding) return nullptr;
    char* result = head + padding;
    head = result + size;
    return result;
  }
};

// RAII helper so a scope's allocations free automatically.
class StackScope {
  StackAllocator& allocator;
  StackAllocator::Marker savedHead;
public:
  explicit StackScope(StackAllocator& a)
    : allocator(a), savedHead(a.mark()) {}
  ~StackScope() { allocator.pop(savedHead); }   // pop on scope exit
};

// Use:
void loadLevel(StackAllocator& frame) {
  StackScope scope(frame);                                 // save marker
  // Construct in place. pop() runs no destructors, so keep scoped types trivially
  // destructible (scope stacks add a finalizer list for the ones that aren't).
  auto* manifest = new (frame.allocate(sizeof(Manifest), alignof(Manifest))) Manifest{};
  parseManifest(manifest);
  loadAssets(frame, manifest);
}                                                          // every allocation in here is freed here

The StackScope destructor is the entire lifetime story. Anything the function allocates from frame is released when the function returns. Anything allocated by a caller's scope before loadLevel is untouched, because the caller's marker is lower in the stack. Unreal's FMemMark is the same RAII marker (cleanup tied to a destructor) over FMemStack[5], and Fredriksson's "Scope Stack Allocation" talk from DICE extends the pattern with a finalizer chain so objects with destructors can live in a scope too[28].

[Back to Memory Allocators from Scratch](index.md)
