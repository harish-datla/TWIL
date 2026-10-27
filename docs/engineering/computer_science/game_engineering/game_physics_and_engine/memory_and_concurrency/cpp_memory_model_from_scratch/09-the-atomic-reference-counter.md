# The Atomic Reference Counter


A texture is referenced by a material on the game thread and by a pending upload on the render thread. Whichever thread drops it last has to free it, exactly once, and only after the other thread's last use. That's a reference counter, the smallest non-trivial lock-free primitive and a favorite engine-interview question. Any owner can drop its hold and the last drop runs the destructor, so the threads have to agree on who is last. The correct version uses a different ordering in each direction:
ref_counted.cpp · the canonical version
Copy

class RefCounted {
  mutable std::atomic<uint32_t> refCount{0};

public:
  void addRef() const noexcept {
    // Relaxed is correct here. We already hold a reference (the caller has a
    // pointer to this object), so the object can't be destroyed underneath us.
    // No ordering with other memory is needed; just bump the counter.
    refCount.fetch_add(1, std::memory_order_relaxed);
  }

  void release() const noexcept {
    // acq_rel on the decrement gives us two things:
    //   - release: every memory access before this release is visible to
    //     the thread that observes refCount == 0 after its own decrement.
    //   - acquire: when our decrement gives refCount == 0, we see every
    //     memory access that other threads did before their releases.
    // Without the acquire side, the destructor could read stale fields.
    if (refCount.fetch_sub(1, std::memory_order_acq_rel) == 1)
      delete this;
  }
};

Worked example

refCount is 2: the material and the upload each hold one reference.

The game thread calls release(). fetch_sub returns the value before the subtraction, 2, and leaves 1. Not 1, so no delete.

The render thread calls release(). fetch_sub returns 1 and leaves 0, so this thread deletes. The test == 1 means "my decrement took it to zero".

The two threads can't both get 1 back, even if they release at the same instant: RMWs on one atomic apply one at a time, so one of them sees 2.

The classic alternative is memory_order_release on the decrement and a separate std::atomic_thread_fence(memory_order_acquire) only on the path that deletes. That drops the acquire half from every decrement that doesn't reach zero, which is most of them. The Boost.Atomic documentation's reference-counting example, written as the hooks for boost::intrusive_ptr, uses this pattern[17]:
ref_counted_optimized.cpp · the Boost variant
Copy

void release() const noexcept {
  // Release is enough on the common path: publish all our writes
  // so the eventual deleting thread can see them.
  if (refCount.fetch_sub(1, std::memory_order_release) == 1) {
    // We are the deleter. The acquire fence pairs with every other
    // thread's release decrement, so their writes are visible here.
    std::atomic_thread_fence(std::memory_order_acquire);
    delete this;
  }
}

Three follow-up questions interviewers like:

    Why relaxed on addRef? The caller already holds a reference, so the object can't be destroyed until after this call, and no other memory needs ordering. Increments don't synchronize with anything.
    What if release used relaxed? A bug. The thread that runs the destructor can see stale values of fields other threads wrote before dropping their references. The release/acquire pairing is what makes those writes visible to it.
    What if everything used seq_cst? Correct. On x86 and ARMv8 a seq_cst RMW compiles to the same instruction as acq_rel (a LOCK XADD, or an LDAXR/STLXR loop or LDADDAL), so the RMWs themselves cost the same; the difference shows up on ARMv7 and POWER, where seq_cst adds a heavier barrier, and wherever a seq_cst load or store replaces a relaxed one.

libstdc++'s std::shared_ptr uses the first version: an acq_rel RMW for the decrement[18]. The weak count is a separate atomic. make_shared puts the object and the control block in one allocation, which saves an allocation but ties the storage to the weak count: the object is destroyed when the strong count reaches zero, but its memory isn't freed until the weak count does too. That matters when objects are large and weak pointers are long-lived.
Check Your Understanding Q1 of 1

You profile a hot loop and find shared_ptr reference counting takes 12% of the frame time on a pre-LSE ARMv8 target (such as the Switch’s Cortex-A57) but 2% on x86. Diagnosis?
ARM atomics are slower than x86 atomics in general.
On x86 each decrement is one LOCK XADD, whose main cost is moving the cache line between cores. On pre-LSE ARMv8 it’s an LDAXR/STLXR retry loop: under contention the store-exclusive keeps failing because another core touched the line, so each decrement can take several round trips.
ARM caches are smaller.
0/1 correct

[Back to The C++ Memory Model from Scratch](index.md)
