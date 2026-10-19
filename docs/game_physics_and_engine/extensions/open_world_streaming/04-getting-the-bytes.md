# Getting the bytes

Read a cell's file on the thread that draws frames and the game freezes for as long as the disk takes. A queued cell's data is read on a worker thread, never on the thread that draws frames. This is the stage most often assumed to be the bottleneck. The File Streaming tutorial built the async read path and the residency pool; the streamer dispatches cell loads onto the job system and is handed back a buffer when the read completes.
The platform async-I/O reality

On Windows, open with FILE_FLAG_OVERLAPPED and drive completions through an I/O completion port (IOCP); on Linux, io_uring batches many reads through one io_uring_enter via its submission/completion rings[5]. In Rust, tokio::fs is not truly asynchronous: it runs ordinary blocking file operations on a spawn_blocking thread pool, because, in Tokio's words, "most operating systems do not provide asynchronous file system APIs"[6]. For real async file I/O on Linux you reach for an io_uring crate. Either way, the bytes arriving is not the same as the cell being usable, which is §5.

Worked example

Say a cell is 40 MB. An NVMe SSD reading roughly 3.5 GB/s delivers it in about 11 ms, on a worker thread, while frames keep drawing. A hard disk at roughly 100 MB/s takes about 400 ms before seeks, which is why slow disks do cause pop-in. On the SSD the read is long finished by the time the cell's turn comes to go live.
