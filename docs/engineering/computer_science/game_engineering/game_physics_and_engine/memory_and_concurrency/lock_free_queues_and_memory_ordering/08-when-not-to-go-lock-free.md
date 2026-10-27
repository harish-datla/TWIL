# When Not to Go Lock-Free


You've profiled a queue, the lock shows up, and you're about to write your own lock-free version. The strongest practical advice in this tutorial: most of the time, don't hand-roll it. Lock-free correctness is subtle on weak hardware, hard to test because races hide, and easy to get wrong in reclamation.

The expert consensus is to use a proven implementation. The C++ Core Guidelines say not to use lock-free programming "unless you absolutely have to"; the alternative CP.100 lists is lock-free structures "implemented by others as part of some library"[12]. In practice: in C++ reach for moodycamel::ConcurrentQueue (MPMC) or ReaderWriterQueue (SPSC, wait-free)[10]; in Rust reach for crossbeam's ArrayQueue (bounded MPMC), SegQueue (unbounded), and crossbeam-deque (the Chase-Lev work-stealing deque, whose owner pushes and pops one end while idle threads steal from the other[13], weak-memory-correct, used under Rayon)[9][14]. The code in this tutorial is for understanding; ship the library.

[Back to Lock-Free Queues and Memory Ordering](index.md)
