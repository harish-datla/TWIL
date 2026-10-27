# std::deque: Chunked, Two-Ended


A replay buffer appends each new event at the back and drops the oldest from the front. With a vector, dropping from the front shifts every remaining element. std::deque ("double-ended queue") gives you push_front and pop_front at O(1), which vector doesn't. It does this by storing elements in a sequence of fixed-size chunks, with a top-level index ("map") of pointers to those chunks[21]. When the front chunk is full, push_front allocates a new chunk and adds it to the map; the existing chunks don't move. Indexing is two loads (chunk pointer, then element) instead of one.

Implementations pick a chunk size, and the picks differ wildly[22]:

    libstdc++: 512-byte chunks. For deque<int>, 128 elements per chunk; for elements of 512 bytes or more, one element per chunk.
    libc++: 4096-byte chunks when the element is smaller than 256 bytes; a fixed 16 elements per chunk above that.
    MSVC: 16-byte chunks, so the element count shrinks with element size: 16 elements for 1-byte elements, 8 for 2-byte, 4 for 4-byte (so deque<int> has chunks of 4), 2 for 8-byte, and one element per chunk for anything larger than 8 bytes. That last regime has the poor cache behavior: a deque of any reasonably sized struct on MSVC is about one heap allocation per element. The chunk size is part of MSVC's ABI, so it can only change at an ABI break.

Worked example

A deque<double> holding 10,000 doubles (8 bytes each).

libstdc++: 512 / 8 = 64 doubles per chunk, so 10,000 / 64 = 156.25, rounded up to 157 chunks. Iterating crosses a chunk boundary 156 times in 10,000 steps.

MSVC: 16 / 8 = 2 doubles per chunk, so 5,000 chunks: 5,000 separate allocations, and a jump to a new address every second element.

Pictured below: a generic layout with 8-element chunks (libstdc++ with 64-byte elements). Push at either end and a new chunk is allocated when the current end chunk fills; existing chunks never move.
Diagram · deque chunk map
push_front ▸
push_back ▸
pop_front ▸
pop_back ▸
Reset ⟲
elements
0
chunks
1
chunk size
8 elems
The top row is the map of chunk pointers and each box below is one 8-element chunk. The deque starts empty in the middle of one chunk: press push_back five times and the fifth push allocates a second chunk (the readout goes to 5 elements, 2 chunks), then push and pop at both ends. Two-level structure: a map of pointers, each pointing to a fixed-size chunk. Iteration walks within a chunk (cache-friendly), then jumps to the next chunk at an unrelated address. push_front never moves existing elements; when the front chunk is full it allocates a new one. The widget frees a chunk once popping empties it, as libstdc++ does. O(1) work at both ends with no element moves is why std::deque is the default backing store for std::queue.

Use it when you need O(1) push and pop at both ends, when reference stability after front/back insertion matters, or when you want to grow without periodic 2× reallocations. Common production uses:

    Single-threaded work queues where new work goes on one end and gets processed from the other (a shared queue across threads needs a lock or a concurrent design).
    Event histories where new events go on the back and stale ones get popped off the front.
    The default backing store of std::queue and std::stack. queue<T> is just queue<T, deque<T>>.

deque is not always cache-friendly

Within a chunk, iteration behaves like a vector: a 512-byte libstdc++ chunk of 8-byte doubles is 8 cache lines swept in order. Each jump to the next chunk lands at an unrelated address, costs an extra load through the map, and can break the prefetcher's stream, so a deque pays an extra penalty once per 64 doubles. That is far better than a list's miss per element and within a small constant of a vector. On MSVC, with one element per chunk for anything larger than 8 bytes, every element is a jump, and the deque behaves more like a list than a vector. For pure-iteration workloads on libstdc++ or libc++, std::vector typically wins by a small constant; for two-ended access, deque wins among the standard containers because the alternative is a vector with O(n) insert(begin()).

[Back to C++ Containers](index.md)
