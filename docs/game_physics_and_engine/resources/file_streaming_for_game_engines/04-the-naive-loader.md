# The Naive Loader


You need to load a level's few thousand assets without freezing the frame, and you want the simplest code that does it. That's a worker thread, a queue of pending reads, and a blocking read() for each one. The caller submits a request and gets back a future (a handle it can wait on for the result); the worker dequeues, reads, and signals.
naive_loader.cpp · the simplest design
Copy

// One worker thread, a queue of read requests, a blocking pread()
// for each one. The caller gets back a future it can wait on.
#include <atomic>
#include <condition_variable>
#include <cstdint>
#include <future>
#include <mutex>
#include <queue>
#include <stdexcept>
#include <thread>
#include <unistd.h>   // pread(); POSIX

struct ReadRequest {
  int                  fileDescriptor;   // already-opened file (returned by open())
  int64_t              byteOffset;       // where in the file to start reading
  size_t               byteCount;        // how many bytes to read
  void*                destination;      // caller-provided buffer to fill
  std::promise<void> completionPromise; // signaled when the read finishes
};

class NaiveLoader {
  // The mutex protects pendingQueue. Held only briefly: just long
  // enough to push or pop a single request.
  std::mutex                          queueMutex;

  // Lets the worker sleep until somebody calls notify_one(), so we
  // don't busy-wait on an empty queue.
  std::condition_variable             requestAvailable;

  // FIFO of pending reads. Producers push at the back, the worker
  // pops from the front.
  std::queue<ReadRequest>            pendingQueue;

  // Destructor flips this to false so the worker exits its loop. Atomic
  // because the worker reads it outside the mutex in its while condition;
  // a plain bool written by one thread and read by another is a data race.
  std::atomic<bool>                   isRunning{true};

  // The single OS thread that drains pendingQueue. One thread = one
  // outstanding read at a time. That's the design's main weakness.
  // Declared last: members initialize in declaration order, so the thread
  // must not start until every member it reads (isRunning) exists.
  std::thread                         workerThread;

public:
  NaiveLoader() : workerThread([this] { workerLoop(); }) {}

  ~NaiveLoader() {
    {
      // Flip the flag while holding the mutex. Flipped outside it, the
      // worker could check the predicate, see "keep running", and go to
      // sleep just after the notify below fires: a lost wakeup, and join()
      // hangs forever.
      std::lock_guard lock(queueMutex);
      isRunning = false;
    }
    requestAvailable.notify_one();      // wake the worker so it can see isRunning
    workerThread.join();
  }

  // Enqueue a read. Returns a future that becomes ready once the
  // worker has serviced this particular request.
  std::future<void> submit(int fileDescriptor,
                            int64_t byteOffset,
                            size_t byteCount,
                            void* destination) {
    ReadRequest request{fileDescriptor, byteOffset, byteCount, destination, {}};
    auto completionFuture = request.completionPromise.get_future();
    {
      // Hold the lock only long enough to push.
      std::lock_guard lock(queueMutex);
      pendingQueue.push(std::move(request));
    }
    requestAvailable.notify_one();        // wake the worker if it's sleeping
    return completionFuture;
  }

  // The worker thread runs this loop until shutdown.
  void workerLoop() {
    while (isRunning) {
      ReadRequest request;
      {
        std::unique_lock lock(queueMutex);

        // Sleep until there's a request to handle, or we're shutting down.
        // cv.wait() drops the lock while sleeping and re-acquires it on wake,
        // so the queue check below is always safe.
        requestAvailable.wait(lock, [&] {
          return !pendingQueue.empty() || !isRunning;
        });
        if (!isRunning) return;
        request = std::move(pendingQueue.front());
        pendingQueue.pop();
      }

      // THE BLOCKING CALL. The worker sits here for ~70 µs (NVMe device read)
      // to ~10 ms (HDD seek + read). The whole point of §5 is to stop blocking
      // here so the device queue can stay full.
      ssize_t bytesRead = pread(request.fileDescriptor,
                                request.destination,
                                request.byteCount,
                                request.byteOffset);
      // A short read (the range runs past end of file) leaves the tail of the
      // buffer unfilled, so only a full read counts as success.
      if (bytesRead == static_cast<ssize_t>(request.byteCount)) {
        request.completionPromise.set_value();
      } else {
        request.completionPromise.set_exception(
            std::make_exception_ptr(std::runtime_error("read failed or came up short")));
      }
    }
  }
};

This works. It compiles, it's about 60 lines of code, and on a fast SSD it will load a level in a few seconds. It's the usual first design, and it doesn't scale beyond level loading, for three specific reasons.
Predict

Before reading the next list: look at the code above and try to name at least one reason it falls far short of 7 GB/s on a 7 GB/s NVMe, especially with small reads. Two for partial credit, three for the full set.
Three things go wrong as the requests get hotter

    Queue depth of one. The worker reads one block, waits for it, then reads the next. The drive never sees more than a single outstanding request, so its internal parallelism (multiple NAND flash channels, multiple DMA engines, which copy data without the CPU) does nothing. The 990 Pro's 1.4M-IOPS rating assumes 16 threads with 32 reads in flight each; at QD1 the same drive is rated at 22K[13].
    Sync calls cross the kernel boundary. Every pread() is a syscall: a user-to-kernel transition, an argument copy, a return, and, for buffered I/O, a copy from the page cache, the kernel's in-memory copy of recently read file data, into the user buffer. The Spectre and Meltdown mitigations made each transition more expensive; the paper introducing Linux's io_uring, an async I/O interface covered in the next section, calls paying system calls on every I/O "a serious slowdown" for that reason[16].
    One worker can't keep up. If the streamer needs 4 GB/s of throughput in 64-KB reads and each read takes about 70 µs, you need ~60,000 IOPS (I/O operations per second). By Little's law (next section) that's at least 4 in flight at all times, and you really want 16-32 in flight to absorb latency variance. One worker doing blocking reads has exactly one in flight.

Each of these has a fix. §5 covers them, and the sections after it layer on compression, residency and priority. The naïve loader stays useful as a correct baseline to measure those against.

[Back to File Streaming for Game Engines](index.md)
