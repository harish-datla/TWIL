# The Memory Model in One Screen


A producer writes a slot, then bumps an index; the consumer sees the new index but reads the slot's old contents. Whether that can happen is decided by the memory ordering on the index. The Memory Model tutorial builds this in full; here is the working subset. Two ideas, kept separate: atomicity (an operation happens all-or-nothing) and ordering (which writes another thread is guaranteed to see, and when).

    relaxed: atomic, but no ordering. Fine for a counter nobody synchronizes on.
    acquire / release: a release store (a store tagged memory_order_release) synchronizes-with, that is, forms an ordering edge with, an acquire load (tagged memory_order_acquire) that reads its value, so everything the writer did before the release becomes visible to the reader after the acquire[3][4]. This is the pair lock-free queues are built on.
    seq_cst: a single total order over all seq_cst operations, on top of acquire/release. The safe default, the most expensive.

Acquire/release does not "make it atomic"

Atomicity and ordering are independent; even relaxed is atomic. And the acquire only synchronizes if it actually reads the value the release wrote, or a value that atomic read-modify-writes appended to that store's release sequence. Reading a stale earlier value forms no happens-before edge.

Worked example

The writer runs data = 42; then flag.store(1, release). The reader runs flag.load(acquire).

If the load returns 1, it read the release store's value, the edge exists, and a following read of data is guaranteed to see 42. If it returns 0, it read an older value: no edge, no guarantee, and the reader must not touch data yet. Make either side relaxed and even the 1 guarantees nothing about data.

[Back to Lock-Free Queues and Memory Ordering](index.md)
