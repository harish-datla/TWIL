# The ABA Problem


A lock-free stack can pass its tests for weeks before a crash dump shows its top pointing into freed memory. ABA is the classic lock-free trap behind that. A CAS reads value A; another thread changes the location A → B → A (often by freeing a node and the allocator handing the same address back); the original CAS then succeeds because the bits match, even though the "A" is now a different object or freed memory: a false-positive CAS.

Three families of fix, and the trap inside the first one:

    Tagged / versioned pointers: pack a counter into spare pointer bits (or use double-width CAS, a CAS over two adjacent words such as x86's CMPXCHG16B) so A-then-A still differs by the tag.
    Hazard pointers: readers publish what they're dereferencing; a writer won't free anything a hazard slot names.
    Epoch-based reclamation: retire freed nodes into epoch buckets and free only after every thread has moved on. Rust's crossbeam-epoch ships this[9].

Tags fix the comparison, not the memory

A versioned pointer stops the CAS from matching a recycled value, but if your thread still holds a raw pointer to a node another thread freed, dereferencing it is undefined behavior (the language promises nothing about what happens) no matter what the tag says. Memory lifetime needs hazard pointers or epochs. The Memory Model tutorial walks the Treiber-stack race that produces ABA in detail; a Treiber stack is the classic lock-free stack that pushes and pops by CAS on its top pointer.

Worked example

Nodes X, Y and Z sit at addresses 0x1000, 0x2000 and 0x3000, with top = 0x1000. Thread A reads top = 0x1000 and next = 0x2000, then is paused. Thread B pops X and Y, frees both, and pushes a new node; the allocator hands it 0x1000 again.

A resumes with CAS(top, expected 0x1000, new 0x2000). Top holds 0x1000, so the CAS succeeds and top now points at Y's freed memory.

With a tag bumped on every successful CAS, A read (0x1000, tag 0), and B's pop, pop and push left (0x1000, tag 3). A's CAS expects tag 0, finds 3 and fails. A 16-bit tag would repeat only after 65,536 more updates.

Step through it. Thread A reads the top, pauses; thread B pops two nodes, frees them, and pushes a new node at the recycled address; A resumes and its CAS wrongly succeeds. Turn on tags: the CAS fails, and one more step runs the retry:
Step
Reset
versioned pointer (tag)
The boxes are the stack, top first, and the line under them says which thread just acted. Step through once with the tag off, then tick the box and step again. Without tags, A's CAS expects X, finds X (the recycled address), and succeeds, leaving the stack top pointing at freed memory. With the versioned pointer on, the tag has moved and the CAS fails, which fixes the comparison, not the lifetime; step once more and A retries against the new top and succeeds. You'd still need hazard pointers or epochs to free X safely.

[Back to Lock-Free Queues and Memory Ordering](index.md)
