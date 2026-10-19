# A Working Scheduler, in Your Language


Everything so far, the worker pool, per-worker deques, stealing and dependency counts, fits in one file. Below is a complete, working scheduler in two languages: modern C++ (20) and Rust. The C++ version is about 230 lines and uses no dependencies beyond the standard library; the Rust version is about 190 lines, also stdlib-only. Both implement the same design: a fixed pool of OS-thread workers, a work-stealing deque per worker, an injection queue for tasks submitted from other threads, dependency tracking with the hold counts from §8, and a wait() that runs tasks until everything submitted has finished. Idle workers sleep on a condition variable. There are no fibers; the callout below lists what else is left out. The C++ deque is the lock-free bounded Chase-Lev design with Lê et al.'s orderings; the Rust deque keeps a mutex per worker for brevity (crossbeam-deque is the lock-free equivalent).

Pick a language. Read the comments; every interesting line has one.
scheduler · pick a language ↓
C++
Rust
Copy

// ───────────────────────────────────────────────────────────
// scheduler.cpp · work-stealing job system with DAG support
// Compile: g++ -std=c++20 -O2 -pthread -c scheduler.cpp
// (a library: link it with a main() like the usage notes at the end)
// ───────────────────────────────────────────────────────────
#include <algorithm>
#include <atomic>
#include <chrono>
#include <condition_variable>
#include <cstddef>
#include <cstdint>
#include <deque>
#include <functional>
#include <initializer_list>
#include <memory>
#include <mutex>
#include <random>
#include <thread>
#include <vector>

namespace mpg {

// 64 bytes is the x86 line size. It's hardcoded because GCC warns when
// std::hardware_destructive_interference_size appears in a header (its
// value can change with -mtune). Apple Silicon lines are 128 bytes.
constexpr std::size_t kCacheLine = 64;

struct Task {
  std::function<void()> body;
  // Starts at 1: the submitter's own hold, dropped at the end of submit(),
  // so no prerequisite can release the task while submit() is still linking it.
  std::atomic<int>      openPrereqs{1};
  std::mutex            dependentsLock;  // guards dependents and done together
  std::vector<Task*>    dependents;      // tasks waiting on this one
  bool                  done = false;    // set once, under dependentsLock
};

// A bounded ring-buffer deque, owned by one worker.
// Owner pushes/pops at bottom; thieves CAS-steal at top.
// Orderings follow Lê et al.'s C11 Chase-Lev deque, minus the resizing.
class Deque {
  static constexpr std::int64_t kCapacity = 1024;             // per worker
  alignas(kCacheLine) std::atomic<std::int64_t> top{0};       // thieves CAS it
  alignas(kCacheLine) std::atomic<std::int64_t> bottom{0};    // only the owner writes it
  // Atomic slots: a slow thief can read a slot the owner is refilling after
  // a wraparound. Its CAS then fails, but the read must not be a data race.
  alignas(kCacheLine) std::atomic<Task*> buffer[kCapacity]{};
public:
  bool push(Task* task) {                                      // owner only
    std::int64_t bottomIndex = bottom.load(std::memory_order_relaxed);
    std::int64_t topIndex = top.load(std::memory_order_acquire);
    if (bottomIndex - topIndex >= kCapacity) return false;     // full
    buffer[bottomIndex % kCapacity].store(task, std::memory_order_relaxed);
    // Release: a thief whose acquire load sees the new bottom also sees the
    // slot, and everything the submitter wrote into *task before pushing it.
    bottom.store(bottomIndex + 1, std::memory_order_release);
    return true;
  }

  Task* popOwner() {                                           // owner only: the fast path
    std::int64_t bottomIndex = bottom.load(std::memory_order_relaxed) - 1;
    bottom.store(bottomIndex, std::memory_order_relaxed);       // reserve the bottom slot
    // Full fence: thieves must see the reservation before we read top, or
    // the owner and a thief could both take the last task. Store-then-load
    // is the one reordering x86 allows, so this is a real MFENCE even there.
    std::atomic_thread_fence(std::memory_order_seq_cst);
    std::int64_t topIndex = top.load(std::memory_order_relaxed);
    if (topIndex > bottomIndex) {                               // empty: undo the reservation
      bottom.store(bottomIndex + 1, std::memory_order_relaxed);
      return nullptr;
    }
    Task* task = buffer[bottomIndex % kCapacity].load(std::memory_order_relaxed);
    if (topIndex != bottomIndex) return task;                   // 2+ left: no thief can reach this one
    // Last task: race any thief for it with the same CAS a thief uses.
    bool won = top.compare_exchange_strong(topIndex, topIndex + 1,
        std::memory_order_seq_cst, std::memory_order_relaxed);
    bottom.store(bottomIndex + 1, std::memory_order_relaxed);   // empty either way
    return won ? task : nullptr;
  }

  Task* steal() {                                              // other threads: the slow path
    std::int64_t topIndex = top.load(std::memory_order_acquire);
    // Pairs with the fence in popOwner(): read top before bottom.
    std::atomic_thread_fence(std::memory_order_seq_cst);
    std::int64_t bottomIndex = bottom.load(std::memory_order_acquire);
    if (topIndex >= bottomIndex) return nullptr;                // empty
    Task* task = buffer[topIndex % kCapacity].load(std::memory_order_relaxed);
    // Claim slot topIndex. Failure means the owner or another thief got it.
    if (!top.compare_exchange_strong(topIndex, topIndex + 1,
            std::memory_order_seq_cst, std::memory_order_relaxed))
      return nullptr;
    return task;
  }
};

class Scheduler {
  std::vector<std::unique_ptr<Deque>> deques;       // one per worker
  std::vector<std::thread>            workers;
  // Only a deque's owner may push to it, so tasks submitted from other
  // threads (the main thread) go here instead. A mutex is fine on this
  // coarse path; a Vyukov MPMC queue (§6) is the lock-free swap-in.
  std::mutex                          injectLock;
  std::deque<Task*>                   injectQueue;
  std::atomic<bool>                   running{true};
  std::atomic<int>                    inFlight{0};  // submitted, not yet finished
  std::mutex                          sleepLock;
  std::condition_variable             wakeSignal;

  // Which scheduler and deque this thread works for; null / -1 elsewhere.
  static thread_local const Scheduler* workerOwner;
  static thread_local int              workerIndex;

public:
  explicit Scheduler(int workerCount = -1) {
    if (workerCount < 0)   // eq. 1: leave one core for the main thread and the OS
      workerCount = std::max(1, (int)std::thread::hardware_concurrency() - 1);
    for (int i = 0; i < workerCount; i++) deques.push_back(std::make_unique<Deque>());
    for (int i = 0; i < workerCount; i++) workers.emplace_back([this, i] { workerLoop(i); });
  }

  ~Scheduler() {                   // call wait() first: workers don't drain on exit
    running.store(false, std::memory_order_release);
    wakeSignal.notify_all();
    for (std::thread& worker : workers) worker.join();
  }

  // Create a task that runs after every task in prereqs, which may still be
  // queued or running. The caller owns the result; delete it after wait().
  Task* submit(std::function<void()> body, std::initializer_list<Task*> prereqs = {}) {
    Task* task = new Task();
    task->body = std::move(body);
    inFlight.fetch_add(1, std::memory_order_relaxed);
    for (Task* prereq : prereqs) {
      // Checking done and linking happen under one lock, so the prereq
      // can't finish in between and miss this dependent.
      std::lock_guard lock(prereq->dependentsLock);
      if (!prereq->done) {
        task->openPrereqs.fetch_add(1, std::memory_order_relaxed);
        prereq->dependents.push_back(task);
      }
    }
    releaseHold(task);             // drop the submitter's hold
    return task;
  }

  void wait() {                    // help run tasks until everything submitted finished
    while (inFlight.load(std::memory_order_acquire) > 0) {
      if (Task* task = findWork(-1)) runTask(task);
      else std::this_thread::yield();
    }
  }

private:
  // Every hold (the submitter's, one per unfinished prereq) is dropped once.
  // Exactly one caller sees the count go 1 -> 0, and only that one enqueues.
  void releaseHold(Task* task) {
    if (task->openPrereqs.fetch_sub(1, std::memory_order_acq_rel) == 1) enqueue(task);
  }

  void enqueue(Task* task) {
    // A worker pushes onto its own deque (LIFO, cache-hot). Other threads,
    // and a worker whose deque is full, use the injection queue.
    bool ownDeque = workerOwner == this && deques[workerIndex]->push(task);
    if (!ownDeque) {
      std::lock_guard lock(injectLock);
      injectQueue.push_back(task);
    }
    wakeSignal.notify_one();
  }

  Task* findWork(int myIndex) {
    if (myIndex >= 0)
      if (Task* task = deques[myIndex]->popOwner()) return task;   // own work first
    {
      std::lock_guard lock(injectLock);
      if (!injectQueue.empty()) {
        Task* task = injectQueue.front();
        injectQueue.pop_front();
        return task;
      }
    }
    // Random victims spread thieves out instead of piling onto one deque.
    thread_local std::mt19937 rng{std::random_device{}()};
    std::uniform_int_distribution<int> pickVictim(0, (int)deques.size() - 1);
    for (int attempt = 0; attempt < (int)deques.size() * 2; attempt++) {
      int victim = pickVictim(rng);
      if (victim == myIndex) continue;
      if (Task* task = deques[victim]->steal()) return task;
    }
    return nullptr;
  }

  void runTask(Task* task) {
    task->body();
    {
      std::lock_guard lock(task->dependentsLock);
      task->done = true;           // from here on, submit() won't link to us
    }
    // done is set, so nothing appends to dependents any more: read it unlocked.
    for (Task* dependent : task->dependents) releaseHold(dependent);
    inFlight.fetch_sub(1, std::memory_order_acq_rel);
  }

  void workerLoop(int myIndex) {
    workerOwner = this;
    workerIndex = myIndex;
    while (running.load(std::memory_order_acquire)) {
      if (Task* task = findWork(myIndex)) { runTask(task); continue; }
      // Nothing to do: sleep until enqueue() signals. The timeout covers a
      // push that lands between findWork() failing and wait_for() starting.
      std::unique_lock lock(sleepLock);
      wakeSignal.wait_for(lock, std::chrono::microseconds(100));
    }
  }
};

thread_local const Scheduler* Scheduler::workerOwner = nullptr;
thread_local int              Scheduler::workerIndex = -1;

} // namespace mpg

// ── usage ────────────────────────────────────────────────
// mpg::Scheduler scheduler;
// auto* a = scheduler.submit([] { /* work */ });
// auto* b = scheduler.submit([] { /* work */ }, {a});  // b runs after a
// auto* c = scheduler.submit([] { /* work */ }, {a, b});
// scheduler.wait();
// delete a; delete b; delete c;   // or recycle them from a pool

What's intentionally missing

This implementation is meant to read clearly, not to be the fastest possible. Prerequisites must be submitted before the tasks that depend on them (the API takes pointers to existing tasks, so a cycle can't be built). Production versions:

    Replace the per-task mutex around dependents with a lock-free list that a finishing task closes atomically.
    Pool tasks instead of new/delete per task. Reinalter's part 2[7] shows the per-thread linear allocator pattern.
    Use fibers (or coroutines) so jobs can wait mid-run. See §9 and Marl's source[22].
    Replace the mutex-protected injection queue with a lock-free MPMC queue (§6), and grow a full deque instead of spilling to it.
    Add priority queues, named threads for non-thread-safe APIs, and profiler hooks.
    Replace the timed sleep with an event count so idle workers neither poll nor miss a wake-up.


[Back to Job Systems from Scratch](index.md)
