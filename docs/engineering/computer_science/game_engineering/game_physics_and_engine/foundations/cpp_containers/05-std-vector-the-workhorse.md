# std::vector: The Workhorse


The list of visible entities this frame, the triangles in a mesh, the bullets in flight: most engine collections are a run of same-typed elements you add to and loop over, and std::vector is the default container for them. Stroustrup's advice in The C++ Programming Language, fourth edition, is "use vector as your default container"[16] and the C++ Core Guidelines repeat it[17]. The reasoning is exactly the cache argument from §1: a contiguous block of memory is the friendliest layout for the CPU, and most workloads don't need more than that.

A std::vector is three pointers (or equivalents) in the object itself: begin, end, and end-of-storage[18]. The actual elements live in a single contiguous heap block. size() is end - begin; capacity() is end_of_storage - begin. Indexing is one pointer arithmetic plus one load. Iteration is a pointer walk that the hardware prefetcher loves.

Worked example

A vector<int> whose block starts at address 0x1000 holds 6 ints and has room for 10, as in the diagram below.

begin = 0x1000, end = 0x1000 + 6 × 4 = 0x1018, end_of_storage = 0x1000 + 10 × 4 = 0x1028.

size() = (0x1018 − 0x1000) / 4 = 24 / 4 = 6. capacity() = 40 / 4 = 10. v[3] is the int at 0x1000 + 3 × 4 = 0x100C.
Diagram · vector in memory
size 6
capacity 10
The three pointers on the left point into one heap block; drag the size and capacity sliders to move end and end_of_storage. size() is the number of constructed elements; capacity() is how much storage is reserved before the next reallocation. The gap between them is "free room"; push_back uses it without allocating.
The four guarantees that matter

    Constant-time random access. v[i] is one pointer add, one load.
    Amortized constant-time push_back. When there's capacity, push_back is one construction plus one increment. When capacity runs out, the vector grows by a constant factor (2× or 1.5×, depending on the library) and reallocates. The amortized cost is O(1) regardless[19].
    Linear-time insert/erase in the middle. Every element after the insertion point has to be shifted by one slot. For trivially copyable elements the library typically does this with one memmove, which runs at close to memory bandwidth; other types are moved one element at a time.
    Contiguous storage, guaranteed. &v[0], v.data(), and v.begin() all point at the same single block. You can pass v.data() to OpenGL, to memcpy, to ::write(). std::vector<bool> is the famous exception (see §20).

The price of the contiguous block is invalidation: every reallocation invalidates every pointer, reference, and iterator into the vector, and inserts that don't grow the vector still invalidate everything from the insertion point to the end. The next two sections take each in turn.

[Back to C++ Containers](index.md)
