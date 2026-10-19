# Under the Call to malloc


The profiler shows one malloc(256) taking 100 microseconds and the next malloc(256) taking 30 nanoseconds. Same size, same call site. The difference is which path through the allocator each call took, and to see the paths you need the picture under a generic malloc. The call consults a data structure the C library has maintained since the program started, and the cost of one call is the cost of finding a fit in that structure, plus a system call when the structure runs dry.

A generic heap is a region of address space, broken into blocks. Each block has a header (size, free/used flag, sometimes a back-pointer for coalescing) and a payload that the caller sees. Free blocks are threaded onto one or more free lists, sorted by size or by address depending on the policy. A call to malloc(n) finds a free block whose payload is at least n bytes, splits off the remainder, and returns the payload pointer. A call to free(p) looks at the header in front of p, marks it free, and tries to coalesce (merge) with the previous and next blocks if they're free, using boundary tags: a copy of each block's size at its end, so a block being freed can find where its left neighbor starts[10].

Three slow-path costs make up the tail of a malloc latency distribution:

    The free-list scan. First-fit walks the list until something fits; best-fit walks all of it; segregated fits index by size class. On a heap with thousands of free blocks, a sequential scan is bounded by the length of the list, not by the size of the request.
    The heap lock. Modern allocators (jemalloc, tcmalloc, mimalloc) keep a thread- or CPU-local cache so most calls never touch a shared lock; the kernel-allocator lineage from Bonwick & Adams 2001[8] uses per-CPU magazines for the same purpose. The shared tier underneath usually still takes a lock, and a spike happens when several threads need it at once.
    The system call. When the heap runs out, the allocator asks the kernel for more address space with mmap or sbrk. The kernel backs and zeroes each page on first touch (a page fault per 4 KB page). One fault costs on the order of a microsecond rather than nanoseconds, so a thousand of them in one frame cost about a millisecond.

Worked example

A streaming system allocates a fresh 4 MB decompression buffer and writes into all of it. The heap has no 4 MB hole, so the allocator maps new pages from the kernel.

4 MB is 4,194,304 / 4,096 = 1,024 pages of 4 KB, and each one faults on its first write. At roughly a microsecond per fault, that's about 1 ms, 6% of a 60 Hz frame, spent before a single byte is decompressed. Reuse the buffer next time and the faults are gone.

Every allocator in this tutorial removes one or more of those costs by giving up some of the generality. A linear allocator removes the free-list scan by removing the free list. A pool removes the size search by fixing the size. A slab removes the constructor cost by handing back objects that are still constructed. Each is a worse general-purpose allocator and a better one for its workload.
A reminder about the runtime

A C++ new expression does two things: it calls operator new (which in the common standard libraries forwards to malloc) and then runs the constructor. The constructor is yours. The malloc half is what this tutorial replaces. A new[] expression for a type with a non-trivial destructor also stores an array cookie, a hidden element count, ahead of the returned pointer, one concrete reason that mixing delete and delete[] (undefined behavior either way) corrupts the heap in practice. None of the specialized allocators in the rest of this page use operator new; they all take raw bytes and let the engine code construct in place with placement new, new (ptr) T(args), which builds an object at an address you supply.

[Back to Memory Allocators from Scratch](index.md)
