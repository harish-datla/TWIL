# Pitfalls and How to Spot Them


A job system that passes every test can still corrupt a lock-free stack now and then, stall a high-priority job behind low-priority work, or break on the first ARM build. These are the failure modes that turn up in real job systems, what they look like, and how to fix them.
The ABA problem

A thread reads value A from a CAS slot, gets preempted, returns to find the slot still says A, and CASes successfully. But while it was away, the value went from A to B and back to A; the slot is no longer the same object, even though the CAS thinks it is[32]. The classic fix is a tag or stamp packed next to the pointer (spare high bits, or a double-width CAS), so two A's a million operations apart are distinguishable. The tag stops the false match but not the read of a freed node's next pointer beforehand, so production stacks pair it with a reclamation scheme such as hazard pointers, RCU, or epoch-based reclamation, or with a node pool that never returns memory to the OS. The Lock-free Queues tutorial walks through the reclamation side.
Live · ABA Problem idle. press Step.
Step ▸
Run ▶
Reset ⟲

Thread 1 wants to pop the top. It reads the top pointer (call it A) and computes the new top. Then Thread 2 races ahead: it pops A, pops B, then pushes a new node that happens to live at the same address as the original A. Thread 1 wakes up. Its CAS sees A again and succeeds. But the next pointer is now stale.
The CAS only compares addresses, not identity. Two distinct stack states can look identical at the comparison site. Tagged pointers add a counter to the high bits so two "A" states a million ops apart are distinguishable.
Spin-loop pitfalls

Don't busy-spin on an atomic without telling the CPU. A tight while (!flag) burns power, starves the SMT sibling, and on Intel cores the exit from the loop triggers a memory-order pipeline flush. Use _mm_pause() on x86 (or __builtin_ia32_pause(), YieldProcessor()) which emits the PAUSE instruction[33]. On ARM use __yield(). If you spin for more than a few microseconds, fall back to sleeping on a futex (a Linux primitive that sleeps in the kernel only when there's contention) or event-count primitive.
spinwait.cpp · spin, then sleep

// Pattern: spin briefly with PAUSE, then fall back to sleeping.
// Spinning pays off when the flag usually flips within a few
// microseconds; past that, the spinning core is wasted.
auto startTime = std::chrono::steady_clock::now();
while (!readyFlag.load(std::memory_order_acquire)) {
  // PAUSE marks this as a spin loop: the CPU hands execution resources to
  // the SMT sibling and avoids a pipeline flush when the wait ends. Its
  // latency varies widely across microarchitectures, so bound the spin by time.
  for (int spins = 0; spins < 64; spins++)
    _mm_pause();

  // Spun too long: give up the core. Park the current fiber on the flag
  // (in a fiber-aware scheduler) or sleep on a futex/event-count primitive.
  auto elapsedMicros = std::chrono::duration_cast<std::chrono::microseconds>(
      std::chrono::steady_clock::now() - startTime).count();
  if (elapsedMicros > 10) {
    parkFiberOrSleepUntilSet(readyFlag);   // returns once readyFlag is set
    return;
  }
}

Priority inversion

A high-priority task is blocked by a resource held by a low-priority task, while a medium-priority task preempts the low one, indefinitely delaying the high one[34]. The Mars Pathfinder mission hit this in 1997. The fix is priority inheritance (boost the holder's priority while the high-priority task waits) or avoiding long lock-holds. In a job system the same shape appears without any mutex: a high-priority job waits on a counter that low-priority jobs must decrement, while a stream of normal-priority jobs keeps every worker busy. Common fixes are to raise the priority of the jobs a waiter depends on, or to have workers pull from lower-priority queues now and then so nothing starves.
Live · Priority Inversion
no inheritance
priority inheritance
Run ▶
Reset ⟲

Three threads: High, Med, Low. Low takes a lock; High arrives and blocks on it; Med wakes and preempts Low, so High keeps waiting until Med is done. Toggle "priority inheritance" to fix it: Low is temporarily boosted to High's priority while it holds the lock, so Med can't get in.
Mars Pathfinder hit this in 1997. The fix was an uploaded patch that turned on priority inheritance for the mutex involved.
Memory ordering bugs

The worst kind: code that looks right on x86 because x86 is total-store-order (its only reordering is a later load passing an earlier store) and forgives most ordering mistakes, but fails on ARM once you ship to a Switch or a phone. Russ Cox's "Hardware Memory Models" essay[35] is a clear walkthrough of why x86 forgives and ARM doesn't. Test on weak hardware, and run TSan (ThreadSanitizer, -fsanitize=thread), which catches data races, including the ones a missing acquire/release causes on ordinary data. When in doubt, use seq_cst and measure; you can relax later if the measurement says it matters.
Live · Memory Reordering on x86 vs ARM
x86 (TSO)
ARM / Apple Silicon (weak)
Run 1000 trials ▶
Reset ⟲
trials
0
r1=1, r2=0 (stale)
0
other outcomes
0

Message-passing test: Thread A stores data = 1, then flag = 1; Thread B loads the flag into r1, then the data into r2. The broken outcome is r1 = 1, r2 = 0: B saw the flag but not the data. x86-TSO keeps both stores and both loads in order, so plain loads and stores never produce it. ARM can reorder either pair, so without release/acquire it does. (x86 does allow one reordering, a load passing an earlier store to a different address, which is why a seq_cst store costs a fence there.) The simulated timings are illustrative, not measured.
If you only test on x86, this code looks correct. The same source compiled for ARM can produce the "impossible" outcome and corrupt state.
Hyperthreading surprise

Two SMT siblings share one core's L1 cache, execution units and TLB, and split its store buffer. Two threads that each saturate the same resources get far less than twice one thread's throughput from sharing a core, and in bad cases (cache thrashing) less than one thread alone[36]. If your engine has eight worker threads and the box has eight physical / sixteen logical cores, prefer to pin to physical cores when possible. Treat the SMT siblings as a bonus lane for asymmetric work (audio next to render is usually fine; two render workers on the same core is usually not).
Live · SMT Contention
both heavy on one core
spread across physical cores
Run ▶
Reset ⟲
per-thread speed
0.55x
L1 contention
high

Each physical core has two SMT lanes that share L1 and the execution units. When two heavy threads land on the same core, they fight over the same resources; the 0.55× per-thread figure is illustrative. At that figure, two heavy threads on one core get 2 × 0.55 = 1.1 cores' worth of work done. Spread them across physical cores and each runs at full speed, close to doubling combined throughput for this kind of load.
If your scheduler doesn't know which logical cores are siblings, the OS may schedule both heavy workers onto one physical core by accident. Pin to physical cores, or measure and adjust.
Check Your Understanding Q1 of 2

You write a lock-free stack. A thread reads the top pointer (A), gets preempted, comes back, and CASes A → B. The CAS succeeds. Some time later your stack is mysteriously corrupted. What’s the most likely cause?
Your atomics use the wrong memory order.
The ABA problem. While the thread was preempted, the top went A → B → A. Your CAS sees A again but the stack has changed underneath it.
You forgot to call std::flush.
0/2 correct

[Back to Job Systems from Scratch](index.md)
