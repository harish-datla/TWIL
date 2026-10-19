# CAS and the Progress Taxonomy


Two workers try to take the next job at the same instant. With no lock, something has to decide which one gets it and tell the other to try again. Compare-and-swap (CAS) is the universal building block: atomically write a new value only if the location still holds the expected one. Loops of CAS are how multiple threads agree without a lock. But "lock-free" is a precise progress guarantee, and three levels get conflated.
The hierarchy (Herlihy & Shavit)

    Obstruction-free: a thread makes progress if it runs alone (others paused). Allows livelock, threads endlessly undoing each other's work, under contention (several threads hitting the same location at once).
    Lock-free: some thread always makes progress system-wide. No livelock; an individual thread can still starve.
    Wait-free: every thread finishes in a bounded number of its own steps, so no thread can starve. For multi-writer structures that guarantee is usually expensive; the SPSC ring in §4 gets it cheaply because each index has one writer.

Source: Herlihy & Shavit, The Art of Multiprocessor Programming[2].

Worked example

A shared job counter holds 5, and threads A and B both want to bump it. Both read 5 and both try CAS(expected 5, new 6). A's lands first: the counter is 6. B's CAS finds 6, not 5, so it fails and hands back the current value. B retries with CAS(expected 6, new 7) and succeeds: 7, no increment lost.

With a plain load and store instead, both would write 6 and one increment would vanish. The retry loop is lock-free: some thread's CAS always succeeds. It isn't wait-free: if other threads keep winning, B can keep retrying.
Two misreadings of the word

"Lock-free means no waiting" is false: it promises the system progresses, not your thread, which can lose a CAS forever. The "lock" is not a mutex (a lock only one thread can hold at a time): it refers to the program locking up. A mutex algorithm isn't lock-free because, as Preshing puts it, once a thread takes the mutex "your worst enemy could simply never schedule that thread again" and everyone stalls[1]. Scope every claim to the operation and the variant: Vyukov's MPMC (next) and moodycamel's queue are commonly called lock-free, yet neither is wait-free.

[Back to Lock-Free Queues and Memory Ordering](index.md)
