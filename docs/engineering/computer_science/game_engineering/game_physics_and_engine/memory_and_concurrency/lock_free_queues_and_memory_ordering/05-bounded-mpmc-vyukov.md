# Bounded MPMC: Vyukov


A job queue has every worker pushing and popping, so no index has a single owner. Many producers and many consumers can't each own an index, so you need an atomic read-modify-write: read, compute and write back as one indivisible step, as CAS does. Dmitry Vyukov's bounded MPMC (multi-producer, multi-consumer) queue, built on CAS, is a common choice, and the Job Systems tutorial uses it[5].

It's a fixed array of cells, each holding a payload plus an atomic sequence number, with two position counters. To enqueue: load the enqueue position, index the cell, and compare its sequence to the position. If they're equal the cell is free, so CAS the position forward and, on success, write the payload and release-store the cell's sequence to mark it ready. Dequeue mirrors this. The cell sequence is loaded with acquire and stored with release; the position counters use relaxed CAS.

Worked example

Four cells, each sequence starting at its own index: 0, 1, 2, 3.

Enqueue at position 0: cell 0's sequence is 0, equal to the position, so the cell is free. The producer CASes the position from 0 to 1, writes the payload and stores sequence = position + 1 = 1, "ready".

Dequeue at position 0: it wants sequence = position + 1 = 1, finds it, CASes its position to 1, reads the payload and stores sequence = position + 4 = 4. That's the enqueue position that maps to cell 0 on the next lap, so the producer at position 4 finds the cell free, and a stale producer still holding position 0 sees 4, not 0, so it reloads the position and tries again.
It is not, strictly, lock-free, and the sequence number is a generation counter

Vyukov says so himself: it is "not lockfree in the official meaning, just implemented by means of atomic RMW operations w/o mutexes"[5]. A producer that claims a slot and then stalls blocks consumers waiting on that slot. And the per-cell sequence isn't a full/empty flag; it's a lap/generation counter. Positions and sequences only ever increase (a 64-bit counter won't wrap in practice), so a stale thread's CAS or sequence check fails instead of matching a recycled value: no separate ABA tag is needed.

[Back to Lock-Free Queues and Memory Ordering](index.md)
