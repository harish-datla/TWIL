# compare_exchange_weak vs strong


A loader claims an empty slot with a single compare_exchange_weak that expects nullptr and stores its request, and gives up if it returns false. On x86 that works. On an ARM core without the ARMv8.1 atomics it now and then returns false when no other thread touched the slot, and the request is silently dropped. Every C++ atomic has two CAS variants, and that loader picked the one that's allowed to fail for no visible reason. compare_exchange_strong returns false only when the current value actually differs from the expected one; compare_exchange_weak may also fail spuriously, even when they match. The reason is hardware:

    x86. CAS is LOCK CMPXCHG, one instruction that never fails spuriously. Weak and strong compile to the same code.
    ARMv7 and ARMv8.0. CAS is built from a load-linked / store-conditional pair, a load that marks the address and a store that succeeds only if nothing disturbed the mark (LDREX/STREX on ARMv7, LDXR/STXR on ARMv8): load, compare in software, then store-conditional. The store-conditional fails whenever the core's exclusive monitor was cleared in between, which happens when another core writes the line (even with the same value), on an interrupt or context switch, or when the line is evicted. The load already saw the expected value, so such a failure is spurious from the CAS's point of view. Weak reports it; strong loops until the store succeeds or the comparison really fails.
    ARMv8.1+ has CAS instructions. CAS, CASA, CASL, and CASAL arrived with the Large System Extensions in ARMv8.1-A: one instruction, no spurious failures, no retry loop[32]. Compilers emit them when targeting ARMv8.1-A or later. For older baseline targets, GCC 10+ and recent Clang on AArch64 Linux default to -moutline-atomics, which calls a small helper that picks LSE or the LL/SC loop at run time.

The rule:

    Use compare_exchange_weak when you already have a retry loop. The classic case is a CAS-loop pop on a stack or queue. A failed CAS updates the expected value, so a spurious failure just costs one more trip around the loop, the same as a real conflict, and you avoid a loop nested inside a loop.
    Use compare_exchange_strong when there's no loop. One-shot attempts (set once if zero, race to install a sentinel) need strong, because a spurious failure would be misread as losing the race.

A correct weak loop:
cas_weak_loop.cpp
Copy

// Atomically multiply a value by 1.1: no instruction does that, so build it
// from a CAS loop.
void growByTenPercent(std::atomic<double>& value) {
  double currentValue = value.load(std::memory_order_relaxed);
  double nextValue;
  do {
    nextValue = currentValue * 1.1;
    // On failure (spurious or real) the CAS writes the current value into
    // currentValue, so the next iteration recomputes from fresh data.
  } while (!value.compare_exchange_weak(currentValue, nextValue,
            std::memory_order_relaxed));
}

Worked example

value holds 100, and two threads call growByTenPercent at once. Both load 100 and compute 110.

Thread A's CAS runs first: it expects 100, finds 100, and stores 110. Thread B's CAS expects 100 and finds 110, so it fails and writes 110 into currentValue. B's loop recomputes 110 × 1.1 = 121 and its second CAS succeeds.

The result is 121 (to floating-point rounding), both growths applied. A spurious failure on ARM would look the same to the loop: one more pass with a fresh value.

The third argument is the ordering on success. The optional fourth is the ordering on failure; when it's omitted, as here, it's derived from the success ordering (the same ordering, except that acq_rel becomes acquire and release becomes relaxed). A failed CAS writes nothing, so its ordering can't be release or acq_rel.

[Back to The C++ Memory Model from Scratch](index.md)
