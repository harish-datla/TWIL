# What's Next


Next in the series: Lock-free Queues. The kernels on this page run on worker threads that hand data to each other, and the queues between those threads are the next bottleneck: a mutex there makes workers wait their turn, and an audio callback can't take a lock at all. The next page builds queues that never block.

Other places to go from here:

    Pick a hot kernel in your engine and rewrite it twice. Once as plain scalar with __restrict and #pragma omp simd, once with intrinsics. Compare the disassembly and the measured frame cost. The gap is the size of the autovec-vs-intrinsics decision in your codebase.
    Read the Intel Intrinsics Guide[9] for the instructions you're using. Each entry gives the instruction it maps to and a pseudo-code description of the lane semantics, and many list latency and throughput for recent Intel cores. The reference is searchable by mnemonic, by intrinsic name, or by feature flag.
    Set up llvm-mca on a small slice of your inner loop. A few minutes of llvm-mca -mcpu=skylake on a 30-instruction kernel shows which port is saturated and where a dependency chain pins the throughput. uica.uops.info is a similar tool with a more detailed front-end model[33].
    Skim Wojciech Muła's blog and Daniel Lemire's blog, at 0x80.pl and lemire.me/blog. Between them they cover hundreds of small SIMD techniques with microbenchmarks on x86 and Arm.
    Pair with the Assembly tutorial and the Memory Model tutorial. The first covers the ports, encodings, and calling conventions under the intrinsics; the second covers atomics and ordering for the multithreaded code that runs these kernels.


[Back to SIMD from Scratch](index.md)
