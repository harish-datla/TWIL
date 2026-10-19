# Why a GPU Is Shaped This Way

A 1080p frame has 2,073,600 pixels, and each usually runs a small shader program. A CPU spends transistors reducing the latency of one thread: caches, branch prediction, and out-of-order execution. A GPU spends them on throughput: thousands of arithmetic lanes run the same program over huge batches of data. It hides memory latency by keeping many work groups in flight and switching to another when one waits.

## SIMT

GPUs use SIMT (single instruction, multiple threads). NVIDIA groups 32 threads into a warp; AMD uses wave32 or wave64 depending on architecture. A warp shares an instruction stream. If lanes branch differently, the hardware executes each path while masking lanes that do not take it.

For a full-screen 1920 × 1080 pass there are 64,800 groups of 32 lanes. If a shader has a cheap 10-instruction path and an expensive 100-instruction path, a warp where four lanes take the expensive branch may execute both paths: 110 instruction steps instead of 10. Divergence cost is serialized paths, not a fixed multiplier.

SIMT is not identical to SIMD. SIMD exposes vector width to code, while SIMT presents scalar-looking per-thread code and the hardware handles lane masking. Never hard-code a warp size of 32; it is vendor- and architecture-specific.
