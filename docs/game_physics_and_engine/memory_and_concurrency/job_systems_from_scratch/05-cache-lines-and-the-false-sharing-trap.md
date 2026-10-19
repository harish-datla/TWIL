# Cache Lines and the False-Sharing Trap


Each worker counts the jobs it ran in its own counter, and the counters sit next to each other in one struct. The workers never touch each other's counter, yet the stats code gets slower as you add workers. When a multi-core program runs slower than the single-core version, common causes are lock contention, oversubscription, memory bandwidth, and false sharing. False sharing is one of the more surprising of these, and one of the easier ones to fix once you know the shape.

Caches don't fetch one byte at a time. They fetch a whole cache line, typically 64 bytes on x86-64 (Intel and AMD, including the Zen 2 silicon in the PS5 and Xbox Series X) and 128 bytes on Apple's M-series chips[12]. Two variables that sit within 64 bytes of each other can end up on the same cache line. Now imagine two threads, each writing to one of those variables. The variables are independent, but the cache line is one unit, and the cache-coherence protocol (a MESI variant on x86) tracks ownership per line[13]:

    Core 0 writes to variable a. The line is loaded into Core 0's L1, marked as Modified.
    Core 1 wants to write to variable b. The line is in Core 0's L1, so Core 1 has to invalidate Core 0's copy, then pull the line over to its own L1.
    Core 0 writes a again, so the line moves back to Core 0's L1.
    This repeats for as long as both threads keep writing.

The line pings between the cores' caches even though the data they touch is logically independent. Hence "false sharing": the sharing looks like sharing only to the cache, not to the programmer.

Worked example

The widget's default run: two threads, 2,000,000 writes each, to two 8-byte counters at offsets 0 and 8 of one struct.

Packed: the writes alternate between cores, so every one of the 4,000,000 writes has to pull the line from the other core first: 4,000,000 line transfers. At on the order of 100 cycles each, that's about 400 million cycles, roughly 0.13 s at 3 GHz, spent moving one line back and forth.

Padded to offsets 0 and 64: each core fetches its own line once and keeps it: 2 transfers.

The demo runs that worst case. The top row has two counters packed adjacently in a struct; both threads write to it. The bottom row has the same counters, padded to live on separate cache lines. Same code, same workload, different memory layout:
Live · False Sharing
iterations 20,00,000
RACE ▶
Reset ⟲
packed: line transfers
···
padded: line transfers
···
writes that wait on the other core
···

JS runs on a single thread, so this is a model of the coherence traffic, not a timing. In the model the two cores' writes strictly alternate, so every packed write has to pull the line from the other core, while the padded layout pays one fetch per core. On real hardware the writes don't alternate so neatly, and the measured penalty depends on the core count, the cache topology, and the prefetchers.
On real hardware, alic.dev benchmarked packed versus padded per-producer indices in a multi-producer queue: the packed layout was slower on every machine tested (Apple M1 Pro, Intel i5-9600K, AMD EPYC, Intel Cascade Lake), and generally fell further behind as producers were added[14]. The size of the penalty depends on the workload, the cache hierarchy, and how aggressively the prefetchers pull neighboring lines.
The fix

Make sure data that is touched by different threads sits on different cache lines. In C++17 the standard provides a portable constant for the right alignment:
padding.cpp · the standard idiom

// std::hardware_destructive_interference_size (in <new>) is the implementation's
// recommended minimum offset between two concurrently written objects.
// It's the toolchain's pick: 64 on x86-64. GCC's generic AArch64 value is 256,
// to cover every Arm line size (Apple Silicon's lines are 128 bytes).
constexpr size_t cacheLineBytes = std::hardware_destructive_interference_size;

// Per-worker statistics, one instance per worker. The owner bumps
// jobsCompleted; a thief bumps jobsStolen when it takes one of this worker's
// jobs, so the two counters are written from different cores.
struct WorkerStats {
  // alignas tells the compiler to start this member on a fresh cache line,
  // inserting padding bytes before it if needed. It also raises the struct's
  // alignment, so neighbouring workers' stats in an array never share a line.
  alignas(cacheLineBytes) std::atomic<int64_t> jobsCompleted;

  // This counter lives on its own cache line too, so a thief's writes
  // here don't invalidate the owner's copy of jobsCompleted.
  alignas(cacheLineBytes) std::atomic<int64_t> jobsStolen;
};
// sizeof(WorkerStats) is now 128 bytes on x86, not 16: a little memory per
// worker to keep two cores from fighting over one line on the hot path.

The alignas declaration tells the compiler to start each member on a fresh cache line, padding with empty bytes if necessary. It costs memory: a struct that "should" be 16 bytes grows to 128. On a hot path that two cores write, those bytes are cheap next to the coherence traffic they remove. One caveat is heap allocation: before C++17, new and std::allocator didn't honor alignments above alignof(std::max_align_t) (usually 16), so a heap array of these structs could still straddle lines. C++17's aligned new fixed that.
A portability footnote

C++17 added std::hardware_destructive_interference_size so you wouldn't have to hardcode 64. GCC 12 and later warn (-Winterference-size) when it is used in a header or a public interface: the value affects struct layout, and it can change with -mtune or the compiler version, which silently breaks ABI between libraries built with different settings[46]. Many engines hardcode alignas(64) for x86 and 128 for Apple Silicon instead; the constant is fine inside a single program built with one set of flags.
Check Your Understanding Q1 of 2

Two atomic counters, a and b, sit next to each other in a struct. Thread A increments a in a tight loop; Thread B increments b in a tight loop. Performance is terrible. You insert char padding[64]; between them and performance jumps 4x. Why?
The padding makes the struct align better.
Without padding, both counters share a cache line. Every increment on one core invalidates the other core’s copy of the line, so the cache-coherence protocol bounces the line between cores. With padding the counters live on separate lines and the bouncing stops.
The padding tricks the CPU into prefetching.
0/2 correct

[Back to Job Systems from Scratch](index.md)
