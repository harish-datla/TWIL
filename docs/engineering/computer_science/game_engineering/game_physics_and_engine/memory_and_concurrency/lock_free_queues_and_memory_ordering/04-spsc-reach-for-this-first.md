# SPSC: Reach for This First


The audio thread's command queue has exactly one writer, the game thread, and one reader, the callback. Single-producer, single-consumer (SPSC: one thread pushes, one thread pops) is the queue to reach for first, and it needs no CAS at all. The reason: each index has exactly one writer. The producer owns the write index, the consumer owns the read index, so no two threads ever contend to modify the same atomic.

Correctness comes from one acquire/release pair per direction. The producer writes the slot, then does a release store of the write index to publish it; the consumer does an acquire load of the write index before reading the slot. Each side reads its own index with relaxed, because nothing else writes it. Boost documents its spsc_queue as "a wait-free single-producer/single-consumer queue"[15], and moodycamel's reader-writer queue makes the same claim: wait-free, no CAS loop, not counting the allocation when enqueue grows the queue[10].
SPSC bounded ring (single owner per index)
C++
Rust

#include <atomic>
#include <cstddef>
#include <type_traits>

template <typename T, size_t Capacity>
class SpscRing {
    static_assert(Capacity > 0 && (Capacity & (Capacity - 1)) == 0,
                  "the & (Capacity - 1) mask needs a power-of-two Capacity");
    static_assert(std::is_trivially_copyable_v<T>,
                  "slots stay live between reuses; pop copies out and never destroys");
    // 64 is the x86 line size; Apple Silicon lines are 128 bytes, so pad to 128 when shipping there.
    alignas(64) std::atomic<size_t> writeIndex{0};   // producer writes, consumer reads
    alignas(64) std::atomic<size_t> readIndex{0};    // own cache line each: no false sharing
    alignas(64) T storage[Capacity];
public:
    bool push(const T& value) {                          // producer thread only
        size_t writePos = writeIndex.load(std::memory_order_relaxed);  // we own it
        size_t readPos = readIndex.load(std::memory_order_acquire);    // pairs with consumer release
        if (writePos - readPos == Capacity) return false;              // full
        storage[writePos & (Capacity - 1)] = value;
        writeIndex.store(writePos + 1, std::memory_order_release);     // publish
        return true;
    }
    bool pop(T& out) {                                     // consumer thread only
        size_t readPos = readIndex.load(std::memory_order_relaxed);    // we own it
        size_t writePos = writeIndex.load(std::memory_order_acquire);  // pairs with producer release
        if (writePos == readPos) return false;                         // empty
        out = storage[readPos & (Capacity - 1)];
        readIndex.store(readPos + 1, std::memory_order_release);       // free the slot
        return true;
    }
};

Worked example

An 8-slot ring after a while: writeIndex = 44, readIndex = 36. The indices never wrap; only the slot number does, through & (Capacity - 1), which is & 7 here.

push computes 44 − 36 = 8 = Capacity, so the ring is full and it returns false. The slot it would have written, 44 & 7 = 4, is the one the consumer reads next (36 & 7 = 4). Once pop stores readIndex = 37, the next push sees 7 in flight and writes slot 4.

The widget runs it. The producer and consumer indices chase around the ring; no lock is ever held, and neither index is ever touched by two threads:
Producer rate
Consumer rate
The circle is the ring's 8 slots; green slots hold published values the consumer hasn't read yet, and the text beside it shows both indices. Drag the producer rate above the consumer rate and watch in flight climb to 8/8. Speed the producer past the consumer and the ring fills, so push starts returning false: backpressure, not a bug. Speed the consumer up and it spins on an empty ring. Each index is written by exactly one side, which is why no CAS and no lock appear anywhere.

[Back to Lock-Free Queues and Memory Ordering](index.md)
