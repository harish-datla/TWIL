# SPSC: A Lock-Free Queue with No CAS


The gameplay thread posts sound events, and the audio thread, which can't afford to block on a lock the gameplay thread holds, plays them. One thread writes, one thread reads, and neither should ever wait for the other. The single-producer/single-consumer (SPSC) ring buffer is built for exactly that, and it's the simplest useful lock-free data structure. Engines use it for command streams from the render thread to the RHI thread (the thread that talks to the graphics API), gameplay-to-audio events, profiler scopes, and telemetry. The producer pushes and the consumer pops on different threads without either blocking the other, and there's no compare-and-swap, because each side is the only writer of its own index.
spsc_ring.cpp · the producer owns writeIndex, the consumer owns readIndex
Copy

template <typename T, size_t Capacity>
class SpscRing {
  static_assert((Capacity & (Capacity - 1)) == 0,
    "Capacity must be a power of two so the mask is fast");

  // One cache line per index. Without the padding, every push and pop would
  // invalidate the other side's line (false sharing, §10).
  alignas(64) std::atomic<size_t> writeIndex{0};   // producer writes, consumer reads
  alignas(64) std::atomic<size_t> readIndex{0};    // consumer writes, producer reads
  alignas(64) T storage[Capacity];

public:
  // Returns false if the ring is full. Called only by the producer thread.
  bool push(const T& value) {
    // Only this thread writes writeIndex, so a relaxed read sees our own last store.
    const size_t currentWrite = writeIndex.load(std::memory_order_relaxed);
    // Acquire on readIndex pairs with the consumer's release store to it, so the
    // consumer's read of a slot happens before we overwrite that slot.
    const size_t currentRead = readIndex.load(std::memory_order_acquire);
    if (currentWrite - currentRead == Capacity) return false;  // full

    storage[currentWrite & (Capacity - 1)] = value;   // non-atomic write to slot

    // Release on writeIndex: the slot write above becomes visible to a
    // consumer whose acquire load reads this new index.
    writeIndex.store(currentWrite + 1, std::memory_order_release);
    return true;
  }

  // Returns false if the ring is empty. Called only by the consumer thread.
  bool pop(T& out) {
    // Only this thread writes readIndex, so relaxed is enough here.
    const size_t currentRead = readIndex.load(std::memory_order_relaxed);
    // Acquire pairs with the producer's release: if we see the new index,
    // we also see the slot contents written before it.
    const size_t currentWrite = writeIndex.load(std::memory_order_acquire);
    if (currentWrite == currentRead) return false;       // empty

    out = storage[currentRead & (Capacity - 1)];        // non-atomic read of slot

    // Release: our read of the slot completes before the producer can reuse it.
    readIndex.store(currentRead + 1, std::memory_order_release);
    return true;
  }
};

The pattern is two release-acquire pairs, one per direction. The producer's release on writeIndex publishes the slot write; the consumer's acquire on writeIndex picks it up. The consumer's release on readIndex tells the producer the slot is free; the producer's acquire on readIndex picks it up. Each side reads the index it owns relaxed, since no other thread modifies it. The indices only ever increase (they aren't wrapped at capacity), so the empty and full checks are a comparison and a subtraction that stay correct even when the size_t counters eventually wrap, and the mask is applied only at the slot lookup.

Worked example

Capacity 16, so the mask is 15. After 21 pushes and 7 pops, writeIndex = 21 and readIndex = 7. Occupancy is 21 − 7 = 14, and the next push goes to slot 21 & 15 = 5. The ring is full when the difference reaches 16.

After the counters wrap, writeIndex has gone past 264 and reads 2, while readIndex is 264 − 5. Unsigned subtraction wraps too, so 2 − (264 − 5) = 7: still the right occupancy.

A 64-bit counter takes about 585 years to wrap at a billion pushes a second, but a 32-bit size_t wraps after 232 ≈ 4.3 billion, about 72 minutes at a million pushes a second. That's why the arithmetic has to survive the wrap. Capacity must be a power of two: then every slot index lines up across the wrap, because 232 and 264 are multiples of the capacity.
What's intentionally missing

This ring supports exactly one producer and one consumer; a second producer would race on writeIndex and lose or corrupt pushes. storage constructs every slot up front and copies values in and out by assignment, so T must be default-constructible and copy-assignable, and popped values stay alive in their slots until overwritten; avoiding that takes raw storage with placement-new and explicit destruction. Each side also reloads the other side's index on every call, where production rings cache it and reload only when the ring looks full or empty. There's no batched push, no backpressure beyond returning false, and no way to wake a sleeping consumer (use a condition variable, or std::atomic::wait in C++20[21]).
Live · SPSC ring visualizer
producer rate 200 Hz
consumer rate 200 Hz
Play ▶
Pause ⏸
Reset ⟲
writes
0
reads
0
occupancy
0/16
rejected pushes
0
The 16 slots run around the ring, with the WRITE cursor outside and the READ cursor inside; filled slots show the pushed value. Press play, then set the producer to 400 Hz and leave the consumer at 200 Hz: the ring fills within a tenth of a second, and from then on about 200 pushes a second are rejected, the difference between the two rates. When the producer outruns the consumer, the ring fills and pushes are rejected (push returns false; the counter tallies them). When the consumer outruns the producer, the ring empties and the consumer spins on the empty check. In production code, rejected pushes call for backpressure and an idle consumer calls for a wakeup mechanism; neither is fixed by growing the ring without bound.

[Back to The C++ Memory Model from Scratch](index.md)
