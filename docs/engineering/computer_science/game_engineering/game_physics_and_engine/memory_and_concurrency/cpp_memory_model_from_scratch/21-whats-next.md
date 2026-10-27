# What's Next


    Next in the series: Memory Allocators. Engines replace malloc with allocators built for each lifetime: bump, stack, pool, buddy, TLSF and slab. The threading ideas from this page come back there in thread-local caches, lock-free free lists and per-thread arenas.
    Pair this with the Job Systems tutorial. With the orderings in hand, re-read its work-stealing deque (§07 and the scheduler code in §10), then read Lê et al. 2013[1] in full, then Gyrling's GDC talk on Naughty Dog's fiber-based job system[39].
    Read the formal models. Sewell et al.'s x86-TSO[2], Maranget, Sarkar, and Sewell's ARM/POWER tutorial[9], and Alglave et al.'s "Herding Cats"[33] turned weak memory from something you could only probe on real hardware into models a tool can check a litmus test against.
    Run a litmus checker yourself. herd7 from the diy7 suite[34] takes a litmus test in plain text and reports which outcomes a given memory model allows; litmus7 compiles the same test and runs it on real hardware. Together they're how the formal models were validated against silicon.
    Build something. Write an SPSC ring, port the Chase-Lev deque, stress-test both with more threads than cores, and read the disassembly on x86 and ARM.


[Back to The C++ Memory Model from Scratch](index.md)
