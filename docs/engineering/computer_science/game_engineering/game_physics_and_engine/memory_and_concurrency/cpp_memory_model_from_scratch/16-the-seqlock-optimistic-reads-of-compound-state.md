# The Seqlock: Optimistic Reads of Compound State


The render thread reads the camera's 4×4 transform every frame while the game thread rewrites it. That's 64 bytes of floats, too big for a single lock-free atomic, and a lock would make the reader wait whenever the writer holds it. Bone poses and animation state with a dozen floats have the same shape. The Linux kernel protects data like the 64-bit jiffies counter and its timekeeping state with a seqlock[29]: a sequence number that is even while the data is consistent and odd while a writer is updating it. Readers never lock. They read the sequence, copy the data, and read the sequence again; if both reads match and are even, the copy is consistent, and otherwise they retry.
seqlock.cpp · optimistic reads, writers never wait for readers
Copy

template <typename T>
class Seqlock {
  alignas(64) std::atomic<uint64_t> sequence{0};
  T payload{};

public:
  // One writer at a time. Sequence goes even -> odd (writing) -> next even.
  void write(const T& value) {
    const uint64_t beforeWrite = sequence.load(std::memory_order_relaxed);  // only we change it
    sequence.store(beforeWrite + 1, std::memory_order_relaxed);             // odd: update in progress
    // Release fence: keeps the payload writes below from becoming visible before
    // the odd number. A release store here wouldn't do it; release only holds
    // back accesses that come before it, not after.
    std::atomic_thread_fence(std::memory_order_release);
    payload = value;                                                        // non-atomic write (see below)
    // Release: a reader that sees the new even number also sees the whole payload.
    sequence.store(beforeWrite + 2, std::memory_order_release);
  }

  // Any number of readers. Retries until it copies a consistent snapshot.
  T read() const {
    for (;;) {
      const uint64_t before = sequence.load(std::memory_order_acquire);
      if (before & 1) continue;                          // writer mid-update: try again
      T snapshot = payload;                              // non-atomic read (see below)
      // Acquire fence: keeps the payload reads above from sinking below the
      // second sequence load. This is the seqlock's load-load barrier.
      std::atomic_thread_fence(std::memory_order_acquire);
      const uint64_t after = sequence.load(std::memory_order_relaxed);
      if (before == after) return snapshot;              // no writer ran meanwhile
    }
  }
};

Worked example

The sequence is 6, so the payload is consistent. The writer stores 7, writes the new matrix, then stores 8.

Reader A loads 6, copies the matrix, and loads 6 again, all before the writer starts. The two match, so A returns its copy.

Reader B loads 6 and starts copying just as the writer stores 7, so its copy can be half old matrix, half new. If the copy picked up any new values, the fences guarantee B's second load returns 7 or 8, not 6, so B throws the copy away and loops. If its next first load sees 7, it loops again; once it sees 8 and a matching second 8, it returns the new matrix.

The writer makes the sequence odd, writes the payload, then makes it even again. A reader that sees an odd number retries. A reader that sees an even number copies the payload and re-reads the sequence; if a writer started in the meantime, the second read differs and the reader retries. The orderings: the final release store publishes the payload to any reader whose first (acquire) load sees the new even number; the writer's release fence keeps payload writes from overtaking the odd number; and the reader's acquire fence keeps its payload reads from sinking below the second sequence load. The kernel's write_seqcount_begin() puts a write barrier after the increment for the same reason[29].

One gap remains. When a reader overlaps a writer, payload = value and snapshot = payload race, and C++ makes any data race undefined behavior, even though the protocol throws the torn copy away. The Linux kernel lives outside the C++ model, and many C++ seqlocks accept the gap because mainstream compilers don't exploit it. Copying through std::memcpy doesn't close it: a concurrent memcpy is still a data race. The portable fix is to make the payload relaxed atomics (std::atomic fields, or std::atomic_ref over plain fields in C++20). With atomic fields the fence-based protocol above is race-free, and Boehm's paper walks through why the fences are needed[30]. P1478 proposes a byte-wise atomic memcpy for exactly this case[40].

Use a seqlock when reads vastly outnumber writes, the payload is too big for a single atomic, and readers can tolerate retrying. Examples: a global config snapshot, a "current frame stats" struct, the latest input state read by animation. It doesn't fit data that readers must consume or modify, such as a queue: a reader only ever takes a copy, and may take it several times.

[Back to The C++ Memory Model from Scratch](index.md)
