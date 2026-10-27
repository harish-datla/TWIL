# The Buddy Allocator


A texture streamer needs blocks of 64 KB, 256 KB and 1 MB, frees them in any order as the camera moves, and needs freed neighbors to merge back into big blocks. A pool serves one size. A buddy allocator serves many, with one rule that makes coalescing on free cheap: every block is a power-of-two size, and every block has a buddy at a fixed place (its offset from the start of the arena, XOR'd with its size). When a block frees, the allocator checks whether the buddy is also free, and if so the two coalesce into a block of the next power of two, recursively. Allocation is the inverse: find the smallest power-of-two block that fits the request, splitting larger free blocks if no suitable one exists. Linux's physical page allocator (__alloc_pages) is a binary buddy[30]; AMD's Vulkan Memory Allocator offered a buddy algorithm for custom pools from v2.2 until v3.0 replaced it with TLSF[18]; BitSquid documented a CPU buddy in 2015[31].

The widget draws the arena as a binary tree: each row is one block size, and a block that has been split shows its two halves in the row below. Pick a request size and allocate; the allocator splits the smallest free block that fits until it reaches the right size. Free random frees one allocated block, and if its buddy is also free the two coalesce back up the tree:
Live · Buddy allocator (256-byte arena, 16-byte minimum)
request size 32 B
ALLOCATE ▸
FREE RANDOM ✕
RESET ⟲
live bytes
0 / 256 B
overallocation
0 B
largest free
256 B
tree splits
0
Each row is one block size, from the 256-byte root down to 16-byte leaves; a split block shows its two halves in the row below. Reset, pick 48 B and press ALLOCATE: two splits, 16 B of overallocation, and a largest free block of 128 B. Then free blocks at random and watch buddies merge back up. A 48-byte request gets a 64-byte block: 16 bytes wasted to internal fragmentation. Request 80 and you'd get a 128. If request sizes were spread evenly within each power-of-two range, the average block would be three-quarters full, which is where the textbook "buddy wastes 25%" figure comes from. Real size distributions aren't that even: Russell's 1977 simulations on three measured distributions show about 30% for binary buddy, and his model predicts about 44%[7][1]. A free block whose buddy is still allocated also can't merge, which is buddy's external fragmentation: allocate several blocks, free a few at random, and the largest free block often stays small while plenty of bytes are free.
Why coalescing is one XOR

A buddy is determined by address. If a block is at offset a from the start of the arena with size s (a power of two, with a a multiple of s), the buddy of that block sits at offset a ⊕ s. So when block B frees, the allocator computes B's buddy address, looks up that address in the free table, and if it's free, merges. The merged block has size 2s and starts at the lower of the two buddy offsets, a & ~s; its own buddy at the next level sits at that offset ⊕ 2s. Coalescing terminates in O(log heap_size) steps in the worst case (no extra work after one merge fails).
buddy.cpp · the buddy address relationship

// Sketch: the Block header, the per-class freeList array, split() and blockAt()
// are left abstract; the offset arithmetic is the part that matters.

// A block at offset `offset` with size `size` (a power of two, offset a multiple of
// size) has its buddy at offset ^ size. Free both and they merge.
size_t buddyOf(size_t offset, size_t size) {
  return offset ^ size;
}

// Round a request up to the smallest power-of-two block that fits.
size_t roundUpPow2(size_t bytes) {
  return std::bit_ceil(bytes);                   // C++20 <bit>: 1 << ceil(log2(bytes))
}

// Free-list index of a power-of-two block: 0 for kMinBlockSize, 1 for twice that, ...
size_t classOf(size_t blockSize) {
  return std::countr_zero(blockSize) - std::countr_zero(kMinBlockSize);
}

// Allocate: take the smallest free block of `size` bytes (a power of two),
// splitting a larger free block on the way down if none exists.
Block* allocate(size_t size) {
  size_t wanted = classOf(size);
  for (size_t i = wanted; i < kMaxClasses; ++i) {
    if (!freeList[i].empty()) {
      Block* block = freeList[i].pop();
      while (i > wanted) {                        // split down to the requested size
        --i;
        Block* upperHalf = split(block);         // halves block->size, returns the upper buddy
        freeList[i].push(upperHalf);
      }
      block->isFree = false;
      return block;
    }
  }
  return nullptr;                                // no free block large enough
}

// Free: merge with the buddy while it is free and whole; repeat up the tree.
void deallocate(Block* block) {
  size_t blockSize = block->size;
  while (blockSize < kMaxBlockSize) {
    Block* buddy = blockAt(buddyOf(block->offset, blockSize));
    // A buddy that is allocated, or split into smaller blocks, can't merge.
    if (!buddy->isFree || buddy->size != blockSize) break;
    freeList[classOf(blockSize)].remove(buddy);
    if (buddy->offset < block->offset) block = buddy;   // merged block starts at the lower half
    blockSize *= 2;
  }
  block->size   = blockSize;                     // the surviving header records the merged size
  block->isFree = true;
  freeList[classOf(blockSize)].push(block);
}

Worked example

An empty 256-byte arena gets a 48-byte request, rounded up to 64. The allocator splits 256 into two 128s, then the lower 128 into two 64s, and hands out the 64 at offset 0: 2 splits, 16 bytes of internal waste, and the upper 128 still free.

A 32-byte request comes next. The 64 at offset 64 splits into two 32s, and the caller gets the one at offset 64.

Its buddy is at 64 ⊕ 32 = 96 (in binary, 1000000 XOR 0100000 = 1100000). When both are free they merge into a 64-byte block at 64 & ~32 = 64, whose own buddy is 64 ⊕ 64 = 0, the block holding the 48-byte request. That block is still allocated, so merging stops there.
Where buddy shines, and where it doesn't

Buddy is the right tool when the request sizes are roughly power-of-two-shaped: 4 KB pages, mip levels of power-of-two textures, variable-size scratch arenas. It's the wrong tool when sizes don't bunch near powers of two, because the internal-fragmentation tax is paid on every allocation. A request for 65 bytes gets a 128-byte block. That's 63 bytes, 49% of the block, locked inside the allocation until it's freed; no other request can use them.
Check Your Understanding Q1 of 2

Your texture streamer has a budget of 256 MB and allocates GPU memory in mips of 4 KB, 16 KB, 64 KB, 256 KB, 1 MB. Which allocator does the budget shape suggest, and why?
A linear allocator, because the budget is bounded and a full reset on each level transition is one instruction.
A buddy allocator, because every request size is exactly a power of two: no rounding waste, and freed mips coalesce straight back into their parent blocks.
A general-purpose malloc, because the request sizes span three orders of magnitude and only it serves any size.
0/2 correct

[Back to Memory Allocators from Scratch](index.md)
