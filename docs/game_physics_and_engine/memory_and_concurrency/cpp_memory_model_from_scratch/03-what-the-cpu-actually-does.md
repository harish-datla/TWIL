# What the CPU Actually Does


You write counter = 5 on one core, and for a while other cores still read the old value. Nothing is lost; the write just hasn't reached them yet. To see why, and why the C++ orderings exist at all, you need the picture of the hardware they're hiding. A modern CPU is roughly the following, per core:

    An L1 data cache, the fastest level of the hierarchy, a few cycles from the core: 32 KB on Skylake and AMD's Zen 3, 48 KB on Intel's Sunny Cove, 128 KB on the Apple M1's performance cores[13].
    A store buffer, a per-core FIFO that holds stores until they can be committed to L1: dozens of entries on recent cores (AMD's Zen 3 has 64[36]). A store leaves the pipeline as soon as it retires (the point where the core commits an instruction's result, in program order); the buffer drains in the background.
    A load buffer that tracks in-flight loads so the core can execute them speculatively and out of order.
    On some weakly ordered designs, a way to acknowledge an incoming "this line is now invalid" message before acting on it, so the message doesn't stall the core. McKenney models this as an invalidation queue[14]; it's one way a core can read a stale value after another core's store has completed. TSO doesn't let any effect like that become visible.

The L1 cache talks to other cores' caches through a cache coherence protocol, the rules that keep every core's copy of the same memory in agreement, typically a variant of MESI, named for the four states a line can be in: Modified, Exclusive, Shared and Invalid. MESI guarantees that for any one cache line (the fixed-size block, usually 64 bytes, that caches store and move), only one core at a time holds a writable copy, and every reader sees the same value. Coherence is why memory_order_relaxed still means something. Even with relaxed, all threads agree on a single order of the writes to each atomic (its modification order); what relaxed gives up is ordering across different memory locations.

The widget below tracks one cache line through the four MESI states as two cores read and write it. The worked example below is one sequence worth clicking through.

Worked example

Both cores start Invalid (I): neither has the line.

1. Core 0 reads. Nobody else has the line, so core 0 fetches it and holds it Exclusive (E).
2. Core 1 reads. Core 0 has it E, so it downgrades, and both end Shared (S).
3. Core 1 writes. Its copy is S, so it first invalidates core 0's copy: core 1 goes to Modified (M), core 0 to I.
4. Core 0 reads again. Core 1 holds it M, so core 1 supplies the data and writes it back, and both end S.

All four steps sent coherence messages between the cores. A fifth step, core 0 reading again, is a hit with no traffic: a core only pays when it touches a line it doesn't hold in a usable state.
Live · MESI cache coherence
Core 0
Read
Write
Core 1
Read
Write
Reset ⟲
M
Modified · dirty, exclusive to this core
E
Exclusive · clean, only this core has it
S
Shared · clean, multiple cores have it
I
Invalid · this core's copy is stale
Each box is one core's copy of the line, with its MESI state; the text under them names the last operation and the message it caused. Press Read and Write on either core, or try the four steps above in order. A read of an Invalid line sends a read request; if no other core holds the line, the reader gets it in Exclusive. If another core holds it Modified, that core supplies the data and writes it back, and both end in Shared. A write to a Shared line first invalidates every other copy, then moves to Modified. Those invalidation round trips are a large part of what an atomic write costs when other cores hold copies of its line[14].

The store buffer is the other half of the picture. When the core writes to memory, the write retires from the execution pipeline into the store buffer almost immediately. From the core's perspective, the write is done. From the cache's perspective, and therefore from every other core's perspective, the write hasn't happened yet. The buffer drains in the background, committing entries to L1 once the cache line is in the right MESI state. Until then, every other core sees stale data.

The visible effects on multithreaded code:

    Store-to-load reordering on one core. A store followed by a load from a different address can complete with the load reading the cache before the store reaches it. The core forwards from its own store buffer, so its own loads see its own stores; other cores don't.
    Store-to-load reordering across cores. The Dekker pattern from §1 (set my flag, then read yours): both threads' stores are still in their store buffers when their loads run.
    Independent reads of independent writes (IRIW). Four cores: two writers, two readers. On hardware that isn't multicopy atomic, meaning a store can reach some cores before others, the two readers can disagree about the order in which the two writes became visible, and POWER does this on real silicon[9]. In C++, only seq_cst on all the accesses forbids it.

Check Your Understanding Q1 of 1

Two threads each increment a single shared std::atomic with memory_order_relaxed a million times each, with no other shared state. After both threads finish, what value does the counter hold?
Some value between 1 and 2,000,000, depending on timing.
Exactly 2,000,000. Relaxed still gives cache coherence and atomicity of the RMW; what it gives up is ordering across distinct atomics.
Undefined behavior. You need seq_cst for counters.
0/1 correct

[Back to The C++ Memory Model from Scratch](index.md)
