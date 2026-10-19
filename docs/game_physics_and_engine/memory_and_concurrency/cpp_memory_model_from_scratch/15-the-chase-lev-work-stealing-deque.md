# The Chase-Lev Work-Stealing Deque


One worker has just split a particle update into 64 jobs and pushed them onto its own queue, and seven other workers are idle. The idle ones should take some of those jobs without a lock that all eight would fight over. The Chase-Lev deque[28] is the standard lock-free deque behind work-stealing schedulers. Each worker thread owns one deque of jobs. The owner pushes and pops at the bottom (LIFO, so the next job it runs is the one it just produced, likely still in cache). Other workers steal from the top (the oldest job, which in divide-and-conquer workloads tends to be the biggest, and the end farthest from the owner). There are no locks: the owner and a thief contend only when one job is left, and thieves claim jobs from each other with a CAS on top.

The 2005 Chase-Lev paper specified the algorithm for sequential consistency. Lê, Pop, Cohen, and Zappa Nardelli's 2013 PPoPP paper[1] gives C11 and ARMv7/POWER versions with the barriers each one needs, and proves the ARMv7 version correct. Even x86 needs one of them: their x86 version keeps a single MFENCE in the owner's pop. Rust's crossbeam-deque, which Rayon uses, follows the paper's orderings[27]. Not every engine scheduler steals work: Naughty Dog's fiber-based job system pulls jobs from three shared priority queues, with no stealing[39].
chase_lev_deque.cpp · the C11 orderings from Lê et al.
Copy

// Job must be trivially copyable (a pointer or small handle): the slots are
// std::atomic<Job> accessed relaxed, because a thief can read a slot the
// owner is overwriting. That thief's CAS then fails and it drops the value.
template <typename Job>
class ChaseLevDeque {
  // top is the thieves' end, advanced by CAS; bottom is the owner's end,
  // written only by the owner. Both count up; a slot is index mod capacity.
  alignas(64) std::atomic<int64_t> top{0};
  alignas(64) std::atomic<int64_t> bottom{0};
  alignas(64) std::atomic<CircularArray<Job>*> storage;   // grows when full

public:
  // Owner only. Push to the bottom.
  void push(Job job) {
    int64_t currentBottom = bottom.load(std::memory_order_relaxed);  // only we write it
    int64_t currentTop    = top.load(std::memory_order_acquire);
    CircularArray<Job>* buffer = storage.load(std::memory_order_relaxed);
    if (currentBottom - currentTop > buffer->capacity() - 1) {        // full: grow
      buffer = growBuffer(buffer, currentBottom, currentTop);
      storage.store(buffer, std::memory_order_release);
    }
    buffer->put(currentBottom, job);                                  // relaxed slot store
    // Release fence: a thief that reads the new bottom also sees the slot.
    std::atomic_thread_fence(std::memory_order_release);
    bottom.store(currentBottom + 1, std::memory_order_relaxed);
  }

  // Owner only. Pop from the bottom (the newest job).
  bool pop(Job& out) {
    int64_t currentBottom = bottom.load(std::memory_order_relaxed) - 1;
    CircularArray<Job>* buffer = storage.load(std::memory_order_relaxed);
    bottom.store(currentBottom, std::memory_order_relaxed);           // reserve the slot
    // Full fence: thieves must be able to see the reservation before we read
    // top. Otherwise the store can sit in the store buffer while a thief reads
    // the old bottom and takes the same job (the SB shape from §4). x86 needs
    // this one too; it compiles to MFENCE or a locked instruction.
    std::atomic_thread_fence(std::memory_order_seq_cst);
    int64_t currentTop = top.load(std::memory_order_relaxed);
    if (currentTop > currentBottom) {                                  // was empty
      bottom.store(currentBottom + 1, std::memory_order_relaxed);     // undo
      return false;
    }
    out = buffer->get(currentBottom);                                  // relaxed slot load
    if (currentTop < currentBottom) return true;   // more than one job left: no thief can get this one
    // Last job: race any thief for it with a CAS on top.
    bool won = top.compare_exchange_strong(currentTop, currentTop + 1,
                   std::memory_order_seq_cst, std::memory_order_relaxed);
    bottom.store(currentBottom + 1, std::memory_order_relaxed);       // empty either way
    return won;
  }

  // Any thread except the owner. Steal from the top (the oldest job).
  bool steal(Job& out) {
    int64_t currentTop = top.load(std::memory_order_acquire);
    // Pairs with the fence in pop(): the owner and a thief can't both miss
    // each other's update, so they can't both take the last job without a CAS.
    std::atomic_thread_fence(std::memory_order_seq_cst);
    int64_t currentBottom = bottom.load(std::memory_order_acquire);
    if (currentTop >= currentBottom) return false;                   // empty
    // Lê et al. use consume here; compilers treat it as acquire anyway.
    CircularArray<Job>* buffer = storage.load(std::memory_order_acquire);
    Job claimed = buffer->get(currentTop);                             // relaxed slot load
    // Claim the job. Failure means another thief, or the owner, got it first.
    if (!top.compare_exchange_strong(currentTop, currentTop + 1,
            std::memory_order_seq_cst, std::memory_order_relaxed))
      return false;
    out = claimed;
    return true;
  }
};

Worked example

top = 10 and bottom = 13: three jobs, at indices 10, 11 and 12.

Owner pops: it sets bottom to 12 and reads top = 10. Since 10 < 12, jobs 10 and 11 still stand between the thieves and job 12, so the owner takes job 12 with no CAS.

A thief steals: it reads top = 10 and bottom = 12, sees a job, and CASes top from 10 to 11. It gets job 10, the oldest.

The last job: now top = 11 and bottom = 12. The owner pops: bottom becomes 11 and top reads 11, equal, so this is the last job. A thief that read bottom before the owner lowered it also sees one job, so both CAS top from 11 to 12, and exactly one of them succeeds.

Each ordering has a job. The release fence in push publishes the slot before the new bottom, the same message-passing pattern as §7. The two seq_cst fences, one in pop between the bottom store and the top load and one in steal between its two loads, give the owner and a thief a consistent view of how many jobs are left: at least one of them sees the other's update, so a contested last job is always settled by the CAS. The owner's side is a store followed by a load of another location, the SB shape, which is why release and acquire can't do this job. A variant that uses a release store of bottom and a seq_cst load of top instead of the fence looks plausible and is broken even on x86: the release store is a plain MOV that can stay in the store buffer past the load, exactly the reordering TSO allows, and two threads can then take the same job.

The listing grows the buffer with a single pointer store. In production the old buffer can't be freed while a thief may still be reading from it, which is its own deferred-reclamation problem (hazard pointers or epochs again); Lê et al.'s code never frees it. CircularArray and growBuffer are left out: an array of std::atomic<Job> with power-of-two capacity, and a function that allocates one twice the size and copies indices top through bottom - 1 across.

[Back to The C++ Memory Model from Scratch](index.md)
