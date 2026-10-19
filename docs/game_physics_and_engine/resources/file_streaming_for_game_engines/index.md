# File Streaming for Game Engines

Your open world is 100 GB on disk and the console has 16 GB of RAM. Textures pop in late, the frame hitches when a file read blocks, and a drive rated at 7 GB/s delivers a tenth of that to your loader. The fix is a streamer: the system that loads what the player is about to need, evicts what they've left behind, and keeps the drive busy without stalling the frame. This page builds one from scratch in C++ and Rust, starting from one blocking read() and working up to async I/O, GPU decompression, priority scheduling and sparse virtual texturing, with a live demo at every stage. It's the same family of techniques that lets Spider-Man swing across Manhattan and Unreal's Nanite render a quarry of statues without a loading screen.
Time
~55 min
Level
Junior engine programmer
Prereqs
You can read C++ or Rust; the Job Systems tutorial isn't required but pairs well
Hardware
Some idea that disks are slower than RAM
◂ Build a Game Engine
Phase 4 · Resources
Next · Asset Pipeline & Serialization ▸
What you'll be able to do by the end

    Estimate what a read costs on each storage tier, and explain why a blocking read loop gets under 100 MB/s from a 7 GB/s drive.
    Keep the drive's queue full with async I/O (io_uring, IoRing, DirectStorage) and size the queue with Little's law.
    Pick a compression stack and say whether the drive or the decompressor limits a load.
    Build a residency pool with an eviction policy and a priority score that decides what loads first.
    Explain how sparse virtual textures and Nanite-style cluster streaming page in only what the camera needs, and read a ~250-line streamer in C++ or Rust.


## Sections

- [Why a Game Engine Needs a Streamer](01-why-a-game-engine-needs-a-streamer.md)
- [A Short History of Getting Fast at I/O](02-a-short-history-of-getting-fast-at-io.md)
- [The Storage Hierarchy and the Cost of a Read](03-the-storage-hierarchy-and-the-cost-of-a-read.md)
- [The Naive Loader](04-the-naive-loader.md)
- [Async I/O: Let the Kernel Do the Waiting](05-async-io-let-the-kernel-do-the-waiting.md)
- [Compression and the GPU Decompression Revolution](06-compression-and-the-gpu-decompression-revolution.md)
- [Bundles: Why One Big File Beats a Million Little Ones](07-bundles-why-one-big-file-beats-a-million-little-ones.md)
- [Residency Pools and Eviction Policies](08-residency-pools-and-eviction-policies.md)
- [Priority: What to Load First](09-priority-what-to-load-first.md)
- [Sparse Virtual Textures](10-sparse-virtual-textures.md)
- [Cluster Streaming: Doing for Geometry What SVT Did for Textures](11-cluster-streaming-doing-for-geometry-what-svt-did-for-textures.md)
- [DirectStorage, PS5, and the Modern Fast Paths](12-directstorage-ps5-and-the-modern-fast-paths.md)
- [A Working Streamer, in Your Language](13-a-working-streamer-in-your-language.md)
- [Try It Yourself](14-try-it-yourself.md)
- [How Unreal Does It](15-how-unreal-does-it.md)
- [How Unity Does It](16-how-unity-does-it.md)
- [Pitfalls and How to Spot Them](17-pitfalls-and-how-to-spot-them.md)
- [Where to Go from Here](18-where-to-go-from-here.md)
- [Sources & Further Reading](19-sources-and-further-reading.md)
