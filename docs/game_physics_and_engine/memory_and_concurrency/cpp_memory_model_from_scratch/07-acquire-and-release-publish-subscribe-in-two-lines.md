# Acquire and Release: Publish-Subscribe in Two Lines


The texture-upload bug from the top of the page is an instance of the most common pattern in lock-free programming: publishing. A producer thread builds an object somewhere in memory, then sets an atomic flag to advertise that the object is ready. A consumer thread polls the flag, and when it observes "ready," it can safely use the object. The producer's release store on the flag synchronizes-with the consumer's acquire load on the flag, and every write the producer did before the release is now visible to the consumer after the acquire.

The pattern is so common it has a name in the formal literature: message passing (MP), the second litmus test from §4. Acquire/release is what makes it work without an explicit lock:
message_passing.cpp · the publication pattern
Copy

// Producer thread:
int sharedData[1024];                       // non-atomic payload
std::atomic<bool> payloadReady{false};

void producer() {
  for (int i = 0; i < 1024; ++i)
    sharedData[i] = computeValue(i);              // non-atomic writes
  payloadReady.store(true, std::memory_order_release);  // the synchronizing store
}

// Consumer thread:
void consumer() {
  while (!payloadReady.load(std::memory_order_acquire))  // the synchronizing load
    std::this_thread::yield();
  // Once the acquire load returns true, every write the producer made
  // before its release store happens before this point. Safe to read.
  int sum = 0;
  for (int i = 0; i < 1024; ++i)
    sum += sharedData[i];                         // non-atomic reads, no race
}

sharedData is a plain non-atomic array, and there's no data race on it: the release/acquire pair puts every producer write before it in happens-before order with every consumer read after it. If either the producer's store or the consumer's load drops to relaxed, the synchronizes-with edge is gone, the reads of sharedData become a data race, and nothing guarantees the consumer sees the writes. On x86 the machine instructions are the same for release/acquire and relaxed, because TSO orders plain loads and stores anyway, but the compiler is still free to move non-atomic accesses across a relaxed store or load. On ARMv8 the difference is LDAR / STLR instead of LDR / STR.

The widget below animates the same pattern as a model. The producer writes a three-part payload and then sets the flag while the consumer polls. The worked example follows one trial in each mode, in the widget's time units.

Worked example

The producer starts at 0.2 and writes payload parts d0, d1 and d2 at 1.2, 2.2 and 3.2. The consumer checks the flag every 0.3, starting at 0.7, and reads the payload 0.05 after it first sees the flag set.

Acquire/release: the flag lands at 4.2, after all three parts. The consumer's check at 4.0 misses it and the one at 4.3 sees it, so it reads at 4.35 and finds all 3 of 3 parts.

Relaxed: the model lets the flag land anywhere from 0.7 to 4.7. Say it lands at 1.6, and the consumer started at 0.6: its checks at 0.6, 0.9, 1.2 and 1.5 miss it, the one at 1.8 sees it, and the read at 1.85 finds only d0. That's a torn read: 1 of 3 parts.
Live · Message-passing race
acquire/release
relaxed
RUN ▶
Pause ⏸
Reset ⟲
trials
0
payload OK
0
torn read
0
The top row is the producer's writes and flag store, the bottom row the consumer's read, marked OK or torn with how many parts it saw. Press run in acquire/release mode and the torn-read counter stays at 0; switch to relaxed and torn reads start to appear. A model, not a CPU trace. With acquire/release, the synchronizes-with edge guarantees the payload is fully written before the consumer reads it, so the torn-read counter stays at zero. With relaxed, the model lets the flag store become visible before some payload writes, and the consumer can see the flag set with the payload half written. ARM hardware can do this; x86 hardware doesn't reorder the stores, but the compiler still may, and with relaxed the payload read is a data race whose result the standard doesn't define[15].
Release-acquire vs release-consume

C++11 also defined memory_order_consume, a weaker form of acquire meant for pointer publication. If you only need ordering for memory reached through the loaded pointer, the hardware's address dependency already provides it on ARM and POWER, so consume could skip the barrier (or the LDAR) that acquire needs. In practice no compiler implemented it as specified, because tracking which later expressions "carry a dependency" through an optimizing compiler proved impractical; GCC, Clang, and MSVC all treat it as acquire[16]. C++17 discouraged its use, and C++26 deprecates it and redefines it to mean acquire[37]. Use acquire.

[Back to The C++ Memory Model from Scratch](index.md)
