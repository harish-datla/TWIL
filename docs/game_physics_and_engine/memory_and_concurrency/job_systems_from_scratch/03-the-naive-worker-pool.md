# The Naive Worker Pool


You have a frame's worth of independent work and eight cores, and you want the simplest way to spread it across them. Start with the simplest thing that could possibly work. N worker threads, one shared queue of work, and a mutex protecting the queue. Each worker loops: grab the mutex, pop a job, drop the mutex, run the job. When the queue is empty, the worker sleeps on a condition variable (a primitive a thread waits on until another thread signals it) until somebody adds work.
naive_pool.cpp · the simplest design
Copy

// A simple thread pool: N worker threads share one job queue,
// protected by a mutex. The usual first design.
class NaivePool {
  // The lock that protects the queue and isRunning. One thread holds it at a time.
  std::mutex                          queueLock;

  // Lets idle workers sleep until somebody calls notify_one().
  // We wake them when a new job arrives or when shutting down.
  std::condition_variable             workAvailable;

  // The queue of jobs. Each job is a callable with no arguments.
  std::queue<std::function<void()>>   jobQueue;

  // One std::thread per worker. They all run workerLoop().
  std::vector<std::thread>            workerThreads;

  // Cleared, under queueLock, by the destructor to tell workers to exit.
  bool                                isRunning = true;

public:
  // Spawn `workerCount` threads. Each enters workerLoop() immediately.
  explicit NaivePool(int workerCount) {
    for (int i = 0; i < workerCount; i++)
      workerThreads.emplace_back([this] { workerLoop(); });
  }

  ~NaivePool() {
    {
      // Clear the flag under the lock: a worker between its check and its
      // wait() would otherwise miss both the flag and the notify.
      std::lock_guard lock(queueLock);
      isRunning = false;
    }
    workAvailable.notify_all();                  // wake every sleeper so it can exit
    for (std::thread& worker : workerThreads) worker.join();
  }

  // Add a job to the queue and wake one sleeping worker.
  void submit(std::function<void()> job) {
    {
      // Take the lock just long enough to push. Releasing it inside
      // the braces lets workers actually pop the job we just added.
      std::lock_guard lock(queueLock);           // every submitter contends here
      jobQueue.push(std::move(job));
    }
    // Wake one worker. If multiple are sleeping, only one needs to run.
    workAvailable.notify_one();
  }

private:
  // Every worker runs this loop until shutdown.
  void workerLoop() {
    for (;;) {
      std::function<void()> nextJob;
      {
        std::unique_lock lock(queueLock);        // every worker also contends here
        // Sleep until there's work to do, or we're shutting down.
        // wait() releases the lock while sleeping and re-acquires it
        // before returning, so the checks below are always safe.
        workAvailable.wait(lock, [this] {
          return !jobQueue.empty() || !isRunning;
        });
        if (!isRunning) return;                  // shutdown: still-queued jobs are dropped
        nextJob = std::move(jobQueue.front());
        jobQueue.pop();
      }                                          // release the lock before running the job
      nextJob();                                 // run outside the lock so other workers can fetch theirs
    }
  }
};

This works. It's about 70 lines of C++ (plus <condition_variable>, <functional>, <mutex>, <queue>, <thread> and <vector>), it's correct, and it gives a real speedup on, say, four cores running 100 ms of mostly-independent work. It is also where most first attempts stop, and it stops scaling quickly. The rest of this tutorial is about why.
Predict

Before reading the next list, look at the code above and try to name the reasons it stops scaling from 4 cores to 32. There are three.
Three things go wrong as you scale

    Lock contention. Every worker grabs the queue's mutex on every job. At 16 workers and a million tiny jobs per second, the lock can become the bottleneck, and the workers spend much of their time waiting in line to look at the queue.
    Cache-line ping-pong. The mutex word, the queue's head and tail, and the condition variable's wait list are written by every push and pop, from every core. Each write needs the cache line holding that data in the writing core's cache, so those lines bounce between cores. §5 shows the mechanism.
    It can't express dependencies. Real frame work has structure: animation runs after physics, render runs after visibility culling. With a flat queue and nothing else, a job that needs another job's result has to block until it finishes, and a blocked thread is a wasted core.

Each of these problems has a fix, and the next sections take them one at a time. By the end the global lock is gone from the hot path, each worker has its own queue so cores rarely touch the same line, and dependencies are first-class.

Worked example

Suppose each pop holds the queue's mutex for 1 µs, counting the hand-off between threads and the cache misses on the queue's lines. The lock serializes every pop, so it can grant at most 1,000,000 per second, however many workers wait for it.

A frame of 10,000 jobs then spends 10 ms passing through the lock one job at a time, 60% of a 16.6 ms frame, before any job does its work. Doubling the workers doesn't change that number; shrinking the jobs makes it worse.


[Back to Job Systems from Scratch](index.md)
