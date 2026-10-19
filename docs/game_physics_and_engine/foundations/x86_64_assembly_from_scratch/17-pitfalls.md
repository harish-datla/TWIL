# Pitfalls


Use this as a checklist when you read or write code at this level. Each item is a mistake that looks fine in the source and shows up as a slowdown or corruption somewhere else:

    Mixing legacy SSE and VEX-encoded AVX. Running non-VEX SSE (movaps xmm0, ...) while the YMM upper halves are dirty from VEX AVX code (vmulps ymm0, ...) costs a state transition of dozens of cycles on Sandy Bridge through Broadwell, and an extra blend μop plus a false dependency per SSE instruction on Skylake and later[3]. The compiler emits vzeroupper before leaving AVX code; hand-written code must too.
    Assuming x86 latency is the cost. Latency only matters on the critical path. A μop with latency 6 and throughput 2-per-cycle can issue every 0.5 cycles when its inputs are independent; the same μop costs 6 cycles per iteration when chained. Read both numbers from uops.info[13], not just the latency.
    Partial-register writes. Writing al and then reading eax or rax stalled for several cycles on P6-family cores (Pentium Pro through Core 2 and Nehalem); Sandy Bridge replaced the stall with an inserted merge μop, and later cores mostly just make the byte write depend on the register's old value[12]. That dependency can still serialize code that looked independent. Treat movzx eax, al as the safe explicit form when widening.
    Clobbering a callee-saved register and not restoring it. The ABI tables in §8 are the contract; ignoring them produces bugs that surface as data corruption many calls away. The defenses are explicit clobber lists in inline asm and a careful read of every hand-written prologue and epilogue.
    Hand-aligning data when the compiler already did. alignas(16) in C++ (_Alignas in C11) gives the compiler the alignment it needs for aligned SSE loads. Hand-aligning by adding pad fields is a steady source of bugs after struct layout changes.
    Optimizing inner loops the compiler already optimized. Before writing intrinsics, look at the compiler's output at -O3 with -march=native. Modern autovectorizers handle the standard patterns: element-wise maps, integer reductions, and floating-point reductions when allowed to reorder them. Hand-rolled intrinsics are right when the compiler bailed out on the pattern, not when it didn't.
    RDTSC doesn't count core cycles. On current chips the TSC ticks at a constant rate regardless of turbo or power states, so it measures elapsed time, not the cycles a kernel took at whatever clock speed it ran. It also isn't serializing: without rdtscp or an lfence, it can execute before the code being timed has finished[29]. Use std::chrono::steady_clock for wall time and the performance counters (perf, VTune, or rdpmc once the OS enables it) for cycle counts.


[Back to x86-64 Assembly from Scratch](index.md)
