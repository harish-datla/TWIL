# Pitfalls


A lock-free queue that passes its tests can still starve a thread, corrupt data on Arm, or run slower than the mutex it replaced. The usual causes:
"Lock-free means no waiting"
It means system-wide progress; a single thread can still starve. Ruling that out is what wait-free adds.
Works on x86, breaks on ARM
A missing acquire/release; TSO hides it, weak memory reveals it. Test on ARM.
CAS on a recycled value
ABA: the bits matched but the object changed. Tag the pointer and reclaim safely.
Tag but still crash
Tags fix the comparison, not lifetime; you still need hazard pointers or epochs.
Slow SPSC ring
head and tail share a cache line, the block caches move as one unit: false sharing, where the line bounces between the two cores. Pad each to its own line.
CAS where you didn't need it
SPSC has one owner per index, so it needs none. Save CAS for true MPMC.
Hand-rolling it
Correctness bugs here survive testing; ship moodycamel/crossbeam unless you can prove the need.

[Back to Lock-Free Queues and Memory Ordering](index.md)
