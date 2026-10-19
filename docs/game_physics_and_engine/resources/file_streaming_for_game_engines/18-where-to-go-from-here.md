# Where to Go from Here


Next in the series is the Asset Pipeline & Serialization: the offline build step that turns source art into the binary formats and bundles this streamer reads, with versioning and hot reload. Past the core pattern, streaming gets engine-specific quickly, and the practical next step is reading other people's implementations and the production talks.
Read these libraries

    DirectStorage SDK samples: Microsoft's github.com/microsoft/DirectStorage includes a GpuDecompressionBenchmark sample you can run locally to see actual numbers on your hardware[38].
    liburing: github.com/axboe/liburing. The reference user-space wrapper around io_uring, maintained by its author. Small and readable, with examples of every submission pattern.
    NVIDIA stdexec: github.com/NVIDIA/stdexec. The C++26 senders/receivers reference implementation, useful for composing async I/O with the rest of your engine's async work.
    O3DE's AZ::IO Streamer: github.com/o3de/o3de. The open-source engine's streamer interface runs async reads with per-request priorities and deadlines, completion callbacks, and rescheduling: a production version of the §13 design you can read end-to-end. Unreal's engine source (free with an Epic account) has the shipping implementations of everything in §15.

Read these papers

    Karis, Stubbe, Wihlidal, A Deep Dive into Nanite Virtualized Geometry, SIGGRAPH 2021[9]. The mesh-streaming reference.
    Barrett, Sparse Virtual Textures, GDC 2008[1]. The texture-streaming reference.
    Megiddo & Modha, ARC: A Self-Tuning, Low Overhead Replacement Cache, FAST 2003[27]. The eviction-policy reference.
    Axboe, Efficient IO with io_uring[16]. The async-I/O reference.
    van Waveren, Software Virtual Textures[2]. The id Tech 5 implementation deep-dive.

Talks

    Cerny, The Road to PS5[7]. Sony's own walkthrough of the PS5 I/O complex and why it was built.
    Ruskin, Marvel's Spider-Man: A Technical Postmortem, GDC 2019[5]. Tile streaming budgeted against HDD read speed at swing speed.
    Ruskin, Streaming in Sunset Overdrive's Open World, GDC 2015[4]. Moving a level-based engine to a streamed open world: hex sizing, duplication for seek-free reads.
    Karis, Journey to Nanite, HPG 2022[39]. Brian Karis's retrospective on the design.
    Guerrilla, Streaming the World of Horizon Zero Dawn[44]. Decima's open-world streaming architecture.

The final exam

Five questions covering the whole tutorial. If you can answer all five without scrolling back, you've got the fundamentals.
Check Your Understanding Q1 of 5

Your game ships a 100 GB world on a 16 GB-RAM console. In one sentence, what is the streamer doing?
Compressing the world’s assets aggressively enough that 100 GB of content fits inside 16 GB of RAM.
Maintaining a resident working set of assets near the player by paging in what the player is about to need and evicting what the player no longer needs, so the live footprint fits in RAM even though the total world does not.
Delegating to the OS’s virtual-memory pager, which swaps cold pages to disk as physical RAM fills up.
0/5 correct

[Back to File Streaming for Game Engines](index.md)
