# Async I/O: Let the Kernel Do the Waiting


Your profiler shows the loader thread mostly idle while the drive delivers 80 MB/s of its rated 7,000. The thread spends its time waiting, one read at a time. Synchronous I/O blocks the thread until the data arrives. Asynchronous I/O hands the kernel a description of what you want, returns immediately, and notifies you later, so one thread can keep dozens of reads in flight. The newer interfaces share a shape: a submission queue (SQ) that you push descriptors into and a completion queue (CQ) that the kernel pushes results onto. Windows' older IOCP has only the completion side.

    Linux: io_uring. Jens Axboe's 2019 design[16]. Two mmapped ring buffers, memory mapped into both the application and the kernel so each side sees the other's writes without a system call. With IORING_SETUP_SQPOLL the kernel polls the submission ring on a dedicated thread; with IORING_SETUP_IOPOLL it polls the storage device for completions instead of taking an interrupt[17]. With submission polling on and the ring kept busy: zero syscalls per I/O.
    Windows (legacy): I/O Completion Ports. The NT-era design, a kernel object that reports each finished read[18]. Open the file with FILE_FLAG_OVERLAPPED; associate the handle with a port; submit reads with ReadFile() and an OVERLAPPED struct; reap completions with GetQueuedCompletionStatus. Still the workhorse on Windows.
    Windows 11: IoRing. An API closely modeled on io_uring, with BuildIoRingReadFile and BuildIoRingRegisterBuffers[19]. Pre-register buffers, submit by integer index, no per-IO pinning.

How many reads do you need in flight? A rule from queueing theory, Little's law, answers it: it relates how many requests are in flight, how long each takes, and how many finish per second.

Worked example

Each read takes 100 µs. With one read in flight at a time, the drive finishes 1 ÷ 100 µs = 10,000 reads a second. To reach 100,000 reads a second at the same 100 µs, you need 100,000 × 100 µs = 10 reads in flight at all times. At 64 KB per read, that's the difference between 0.66 GB/s and 6.6 GB/s.
eq. 1 · Little's law for I/O throughput = queue depth ÷ latency

In words: reads finished per second equal the number of reads in flight divided by how long each one takes. Synchronous I/O pins the queue depth at one, so its throughput can never beat 1 ÷ latency. Async I/O lifts that ceiling by submitting many requests before any of them complete. Hover any term to see what it stands for.

Below is a Linux io_uring submission for a single read, using the liburing helper library. Windows' IoRing follows the same pattern with different function names. IOCP has no submission ring (each read is its own ReadFile call), but its completion side works the same way.
io_uring_read.c · one async read

// 1. Set up the ring once. The kernel allocates both shared
// rings (submission and completion) and maps them into our address space.
struct io_uring ring;
io_uring_queue_init(
    256,                       // queue depth: up to 256 outstanding reads
    &ring,
    IORING_SETUP_SQPOLL);      // kernel polls the submission ring; no syscall per submit
                               // (before Linux 5.11, SQPOLL also needed registered files and privileges)

// 2. Pre-register the destination buffer. Registration pins and maps its
// pages once, so the kernel skips that work on every read.
struct iovec destinationBuffer = {
    .iov_base = destinationPtr,    // where the bytes will land
    .iov_len  = 65536             // 64 KiB max read into this buffer
};
io_uring_register_buffers(&ring, &destinationBuffer, 1);

// 3. Per read: grab a Submission Queue Entry, fill it in, submit.
// get_sqe returns NULL when the ring is full; a real loop would reap
// completions and retry instead of proceeding.
struct io_uring_sqe* submissionEntry = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(
    submissionEntry,
    fileDescriptor,
    destinationPtr,
    65536,                     // bytes to read
    fileOffset,
    /*registeredBufferIndex=*/0); // matches the index we registered above
submissionEntry->user_data = (uintptr_t)myRequestId;  // tag so completions can be matched back to requests
io_uring_submit(&ring);        // under SQPOLL: publishes the new tail, and only makes a
                               // syscall if the kernel thread went idle and needs waking

// 4. Later, drain Completion Queue Entries (CQEs). Each CQE carries the
// user_data tag back so we know which request just finished.
struct io_uring_cqe* completionEntry;
while (io_uring_peek_cqe(&ring, &completionEntry) == 0) {
  RequestId requestId = (RequestId)completionEntry->user_data;
  int       resultCode = completionEntry->res;    // bytes read, or negative errno on failure
  on_complete(requestId, resultCode);
  io_uring_cqe_seen(&ring, completionEntry);   // release the slot so the kernel can reuse it
}

Step through the rings with the widget. The application pushes submission entries (yellow); the kernel's polling thread picks each one up almost at once, which frees its slot, and the read is then in flight at the device (purple). When the device finishes, the kernel writes a completion entry (blue) for the application to reap. The two rings move independently: you can submit 32 reads before any of them complete, and they can complete in any order.
Live · io_uring rings
target queue depth 16
device latency (µs) 70
Run ▶
Pause ⏸
Reset ⟲
submitted
0
completed
0
IOPS
···
The top ring is the submission queue, the middle row is reads in flight at the device, and the bottom ring is the completion queue. Press Run at a target queue depth of 1, then again at 16. At depth 1 and the default 70 µs latency the IOPS stat settles around 13,000, close to 1 ÷ latency = 14,286 (the model adds a short wait in the submission ring and varies each read's latency); at depth 16 it reads around 215,000, about 16 times as many. A real device eventually hits its internal hardware limit, which this model doesn't simulate. Device time is slowed 8,000× so the rings stay watchable; the IOPS stat is computed in device time. The submission ring has 32 slots; depths above 32 work because entries leave the ring as soon as the kernel starts them.
Live · Sync vs Async
synchronous
asynchronous (QD 16)
RUN ▶
Reset ⟲
elapsed
10.2 ms
throughput
0.8 GB/s
Each bar is one read on a timeline; run both modes and compare the elapsed stat. Both modes do the same 128 reads of 64 KB on a modeled 7 GB/s drive with 80 µs latency, animated at the same time scale. Synchronously every read pays the full latency in turn: 128 × 80 µs reads "10.2 ms" and "0.8 GB/s". With 16 in flight the latencies overlap and the run is bounded by the drive's bandwidth instead: "1.1 ms" and "7.0 GB/s".
mmap is not the answer

It is tempting to mmap() the asset file and let the OS demand-page. For a streamer's high-churn working set, don't. Crotty, Leis, and Pavlo's CIDR 2022 paper "Are You Sure You Want to Use MMAP in Your Database Management System?"[20] documents three performance problems that carry over from database workloads to any mapping under constant eviction pressure: page-table contention under concurrent access, single-threaded eviction in the kernel, and TLB shootdowns, where the kernel interrupts other cores to flush their cached address translations. Read-mostly mappings of small, stable data are fine; for the streaming path itself, use explicit asynchronous reads, which give you control over latency and ordering, and for buffered reads use posix_fadvise hints to steer what the page cache keeps[46].
Check Your Understanding Q1 of 2

What does "zero syscalls per I/O" mean in the context of io_uring?
The storage device completes requests so quickly that the kernel never has time to charge a syscall to the calling thread before the data has already landed in the buffer.
With IORING_SETUP_SQPOLL a kernel thread polls the submission ring directly, so submitting a request is a memory write rather than a syscall. Completions land in a ring the application reads directly, so reaping finished requests doesn’t need a syscall either.
The program is statically linked, so reads compile down to inlined kernel transitions with no libc wrapper call appearing anywhere in the profile.
0/2 correct

[Back to File Streaming for Game Engines](index.md)
