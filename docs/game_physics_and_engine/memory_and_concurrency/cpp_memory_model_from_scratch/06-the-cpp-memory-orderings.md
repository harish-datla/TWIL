# The C++ Memory Orderings


Every std::atomic load, store and read-modify-write (RMW, an operation like fetch_add that reads and writes in one indivisible step) takes an optional std::memory_order argument, seq_cst if you leave it out. There are six values. Five matter in practice; the sixth, consume, is covered at the end of §7. acquire applies to loads, release to stores, and acq_rel to RMWs; relaxed and seq_cst apply to all three.

One term from the standard runs through the table. A release store synchronizes with an acquire load of the same atomic when the load reads the value the store wrote. When it does, everything the storing thread did before the store is visible to the loading thread after the load. It's the basic mechanism by which one thread's ordinary writes become safe for another thread to read:
Ordering	Use on	What it gives you	What it costs on x86 / ARM
memory_order_relaxed 	load, store, RMW 	Atomicity and modification-order consistency on this atomic. No ordering across other atomics or non-atomics. 	x86: plain MOV. ARM: plain LDR / STR.
memory_order_acquire 	load, RMW 	If this load reads the value a release store wrote, it synchronizes-with that store. No later load or store can move ahead of this load. 	x86: plain MOV (acquire is free). ARMv8: LDAR (or LDAPR on ARMv8.3+ targets).
memory_order_release 	store, RMW 	This store synchronizes-with an acquire load on the same atomic that reads the value it wrote. No earlier load or store can move past this store. 	x86: plain MOV (release is free on TSO). ARMv8: STLR.
memory_order_acq_rel 	RMW only 	The RMW is both acquire (on the value it read) and release (on the value it wrote). 	x86: LOCK-prefixed RMW. ARMv8.0: LDAXR / STLXR loop. ARMv8.1+: one LSE instruction (e.g. LDADDAL).
memory_order_seq_cst 	load, store, RMW 	Everything acquire/release gives, plus one total order over all seq_cst operations that every thread agrees on. That order is what forbids the SB (0, 0) and IRIW outcomes, which acquire/release alone allow. 	x86: load is a plain MOV; store is XCHG (or MOV + MFENCE). ARMv8: LDAR / STLR, the same instructions as acquire/release; the price is that an LDAR can't complete ahead of an earlier STLR. ARMv7: DMB ISH barriers around plain loads and stores.

Two things to take from the table. First, an ordering is a property of the operation, not of the atomic. The same std::atomic<int> can be loaded relaxed by one thread and with acquire by another; only the acquire load synchronizes. Second, a fence never synchronizes on its own: it needs an atomic store and an atomic load on the same object to carry the edge. A release fence followed by a relaxed store synchronizes with an acquire load that reads that store, or with a relaxed load followed by an acquire fence ([atomics.fences]); a fence with no atomic accesses around it orders nothing across threads.

Worked example

Producer: data = 42; then a release fence, then flag.store(1, relaxed). Consumer: flag.load(relaxed), and if it returned 1, an acquire fence, then read data.

If the consumer's load returns 1, it read the producer's store, so the two fences synchronize and the read of data sees 42. If it returns 0, nothing synchronized, and the consumer correctly skips data.

Delete the flag store and load and keep both fences: there's no atomic value for the consumer to read, nothing synchronizes, and reading data becomes a data race.


[Back to The C++ Memory Model from Scratch](index.md)
