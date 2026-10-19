# False Sharing and the Cache Line


Each worker thread counts the jobs it ran in its own atomic counter, and the counters sit side by side in one struct. No thread ever touches another thread's counter, yet adding workers makes the counting slower. The cause is that cache coherence works at the granularity of a cache line, not an individual address: 64 bytes on x86-64 and most ARM cores, 128 bytes on Apple M-series and POWER[19]. When two atomics live on the same line, a write to either one invalidates the whole line in the other cores' caches, even though the threads never touch each other's variable. The line ping-pongs between cores, and each write can pay a coherence round trip of roughly 50 to 200 cycles[14]. The threads share no data, only the line, which is why this is called false sharing.

The fix is to put each independently-accessed atomic on its own cache line, by padding the struct so the next field lands on the next line:
false_sharing.cpp · what NOT to do
Copy

// Both atomics land on the same 64-byte cache line. Every increment by
// one thread invalidates the line in the other thread's cache.
struct Counters {
  std::atomic<uint64_t> producerCount;  // thread A writes
  std::atomic<uint64_t> consumerCount;  // thread B writes
};

false_sharing_fixed.cpp · with alignas
Copy

// Each atomic starts its own 64-byte cache line, so writes to one never
// invalidate the other. (Use 128 on Apple M-series; see below.)
struct Counters {
  alignas(64) std::atomic<uint64_t> producerCount;
  alignas(64) std::atomic<uint64_t> consumerCount;
};

// Or wrap the atomic in a type that fills a whole line, so every
// array element or member of this type gets a line to itself:
struct alignas(64) PaddedAtomic {
  std::atomic<uint64_t> value;
  char pad[64 - sizeof(std::atomic<uint64_t>)];
};

Worked example

In the first Counters, the two 8-byte atomics sit at offsets 0 and 8, so they share a line unless the struct happens to start in the last 8 bytes of one. With alignas(64) they sit at offsets 0 and 64, on separate lines, and sizeof(Counters) grows from 16 to 128 bytes.

The widget's cost model at 2 threads, on a notional 3 GHz core (3,000,000 cycles per ms): padded, each thread runs a 20-cycle increment in parallel, 3,000,000 / 20 = 150,000 per ms each, 300,000 in all. Packed, increments go one at a time, and half the time the last writer was the other core, so each pays 20 + 100 × ½ = 70 cycles: 3,000,000 / 70 ≈ 42,857 per ms. Padding wins 7×.

At 8 threads the last writer is another core 7 times in 8: 20 + 100 × 7/8 = 107.5 cycles, 27,907 per ms packed against 1,200,000 padded, 43×.

The widget below is a cost model of that benchmark, not a measurement: each thread increments its own counter, either packed onto one shared line or padded onto lines of their own. The model treats the shared line as something only one core can write at a time. Change the thread count and run it:
Live · False sharing cost model
threads 2
RUN ▶
Reset ⟲
packed ops/ms
···
padded ops/ms
···
padded speedup
···
The two bars are total increments per millisecond with the counters packed on one line and padded onto their own lines. Set the thread count and press run: padding wins 7× at 2 threads and 43× at 8. A cost model at a notional 3 GHz, not a hardware measurement. Padded: every thread runs in parallel on its own line at about 20 cycles per locked increment. Packed: only one core can write the shared line at a time, and an increment also pays about 100 cycles to pull the line over whenever another core wrote it last, so total throughput falls as threads are added. Both costs are rough orders of magnitude. This is a worst case: real cores often complete several increments before losing the line, so measured penalties are smaller, but they grow with thread count the same way. alic.dev measured a packed versus padded layout of a lock-free queue on four CPUs and found the packed one slower in almost every configuration, increasingly so as producer threads were added[20].

C++17 added a standard constant for the padding size:
portable_alignment.cpp

// std::hardware_destructive_interference_size is the implementation's
// suggested minimum distance between objects to avoid false sharing.
// x86-64 toolchains report 64; its AArch64 value depends on the compiler.
struct Counters {
  alignas(std::hardware_destructive_interference_size)
    std::atomic<uint64_t> producerCount;
  alignas(std::hardware_destructive_interference_size)
    std::atomic<uint64_t> consumerCount;
};

The constant is fixed at compile time, and its value depends on the compiler, its version, and the tuning target. On x86-64 it's 64. For generic AArch64 tuning, GCC 12 and later report 256 (the top of the range of line sizes it tunes for), Clang 21 reports the same, and Clang 19 and 20 reported 64. GCC's documentation notes that the value follows -mtune and advises against using it anywhere ABI stability matters, such as a library header[38], so many teams hard-code alignas(64) or alignas(128) per platform instead. Lemire measured the effective line size with a strided-copy benchmark in 2023: 64 bytes on an Intel server, 128 bytes on an Apple M2[19].

[Back to The C++ Memory Model from Scratch](index.md)
