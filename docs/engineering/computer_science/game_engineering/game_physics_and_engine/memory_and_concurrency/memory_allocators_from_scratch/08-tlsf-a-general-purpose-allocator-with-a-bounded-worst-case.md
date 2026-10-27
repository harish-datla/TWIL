# TLSF: A General-Purpose Allocator with a Bounded Worst Case


Some allocations fit none of those shapes. Decompression buffers range from 3 KB to 3 MB, live anywhere from a few frames to a few minutes, and free in any order. That's general-purpose work, but the frame budget still needs a worst case you can count. Linear, stack, pool, and buddy each give up some of the generality of malloc to win speed and a bounded worst case. TLSF (Two-Level Segregated Fit) goes the other direction: keep the generality of a free-list allocator (any size, any lifetime, no special workload assumption) but index the free lists with a two-level bitmap that hardware bit-scan instructions search without a loop. The result is an O(1) allocator with a bounded worst case: the 2008 paper measures at most 160 instructions for malloc and 176 for free on x86[17]. Unity's dynamic heap allocator uses TLSF[4], AMD's VMA (Vulkan Memory Allocator, a widely used library for sub-allocating GPU memory) defaults to it since v3.0[18], and MorphOS 2.0 and later use it as the default system allocator[32].

Every free block is bucketed by its size into a first-level class indexed by its power-of-two bucket, and a second-level class that linearly subdivides that bucket into 2SLI sub-buckets (shipping implementations typically pick SLI = 4 or 5; the widget below uses SLI = 2 for visual density). Each (first, second) pair has its own free list. Two bitmaps record which lists are non-empty: FL_bitmap has one bit per row, SL_bitmap[fl] one bit per sub-bucket in row fl.

The search below finds the free list to pop for a request in class (fl, sl), using only masks and bit-scans, so its cost doesn't grow with the number of free blocks:
TLSF find-fit sl′ = ffs( SL_bitmap[fl] & mask≥sl ) ;   if none:  fl′ = ffs( FL_bitmap & mask>fl ) , sl′ = ffs( SL_bitmap[fl′] )

In words: (fl, sl) is the request's class after rounding it up to the next sub-bucket boundary, so every block on the chosen list is big enough. The search first looks in the request's own row: SL_bitmap[fl], masked to sub-buckets at or above sl. If that row is empty from there up, FL_bitmap, masked to rows above fl, names the next power-of-two row with any free block, and that row's lowest non-empty list is taken. At most two ffs bit-scans (find-first-set: the position of the lowest set bit, one instruction) and two masked ANDs, plus the size-to-class mapping: a few dozen instructions with no loop, however full the heap is.

The widget below draws each row of the grid as one first-level class, so the lit cells in a row are that row's SL bitmap, and the green column on the right is the FL bitmap. Pick a request size and allocate: the dashed box marks where the search starts, the yellow box marks the list it pops, and the line under the grid traces each step:
Live · TLSF bitmap (FL = 8 classes, SL = 4 per class)
request size 96 B
ALLOC ▸
FREE RANDOM ✕
SEED HEAP
RESET ⟲
picked (fl, sl)
···
overhead this alloc
···
free blocks
11
live bytes
0 B
Rows are first-level classes, columns the four sub-buckets, lit cells hold free blocks, and the green column on the right is the FL bitmap. The heap starts seeded; set the size to 100 and press ALLOC: the search starts at (2, 3), pops the 120-byte block there, and reports 20 B of overhead, as in the worked example below. A 96-byte request starts in the FL=2 row (64 to 127 bytes) at the SL=2 column (the third of four sub-buckets, 96 to 111 bytes). A 100-byte request is first rounded up to 112, so its search starts at SL=3 and it receives 112 bytes; that rounding is the "overhead this alloc" stat, and it is less than one sub-bucket width (plus any leftover under the 16-byte minimum block, which stays attached). The cost of one alloc is bounded: the class mapping, at most two bit-scans, and the split of any leftover bytes back onto a free list. Freeing returns a block to its class list; real TLSF first merges it with free physical neighbours, which this widget skips because it tracks sizes, not addresses.
The size-class encoding

The search needs a request's row and column first. For a request size n, a smallest block size 2FLI (16 bytes in the widget, so FLI = 4), and a chosen SLI (number of second-level bits), the mapping below turns a size into those class indices:
TLSF class indices f = ⌊log2(n)⌋ ,    fl = f − FLI ,    sl = ⌊ (n − 2f) / 2f − SLI ⌋

In words: f is the power-of-two range the size falls in, fl is that range counted from the smallest block size, and within the range [2f, 2f+1) the second-level index sl is the position of n in that range, scaled to 2SLI sub-buckets. With SLI = 4, each power-of-two range splits into 16 sub-buckets: the 1024..2048 range gets sub-buckets at 1024, 1088, 1152, ..., each 64 bytes wide. The search rounds a request up to the next sub-bucket boundary, adding less than 2f − SLI bytes, which is under 1 / 2SLI of the request: at most 6.25% with SLI = 4, 3.1% with SLI = 5. The original TLSF hands the rounded size to the caller, so that rounding is its internal fragmentation[17]; Conte's implementation rounds only for the search and splits the block to the exact request[33].

Worked example

A 100-byte request with the widget's SLI = 2 and FLI = 4. f = ⌊log2 100⌋ = 6 (100 is between 64 and 127), so fl = 6 − 4 = 2. The row's four sub-buckets are 26−2 = 16 bytes wide, and sl = ⌊(100 − 64) / 16⌋ = 2, the 96 to 111 bucket.

A block from that bucket might be only 96 bytes, so the search rounds the request up to the next boundary, 112, and starts at (2, 3). In the widget's seeded heap that list holds a 120-byte block. The 8 bytes left over are under the 16-byte minimum block, so they stay attached: the caller gets 120 bytes for a 100-byte request, the 20 B the widget reports.

With SLI = 4 the sub-buckets in that row are 4 bytes wide, 100 is already on a boundary, and the rounding adds nothing.
What the bitmap buys you

The bitmap is what makes TLSF O(1). Compare to a segregated free list with the same size classes but no bitmap: finding the next non-empty class requires a linear scan over class indices, which is O(N classes). On a 64-class heap that's 64 indirect-load comparisons. The bitmap collapses all of them into one bit-scan instruction. The 2008 paper's measured worst case is 160 instructions for malloc[17]. That count belongs to the code, not the CPU, and it stays flat as the heap fills, which is the property a frame budget needs.

Matt Conte's mattconte/tlsf on GitHub is a widely used open-source implementation: BSD-licensed, about 1,300 lines of C, one word of header per allocated block (4 bytes on 32-bit targets, 8 on 64-bit), and about 3 KB of control structure[33]. VMA carries its own C++ TLSF implementation; Unity's is closed-source.

[Back to Memory Allocators from Scratch](index.md)
