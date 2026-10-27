# Pitfalls


A SIMD kernel that passes every test on the dev machine can still crash at boot on a player's CPU, produce different bits on another platform, or slow down sharply once the particles it updates come to rest. The common causes:

    Mixing legacy SSE encodings with VEX-encoded AVX. Running non-VEX SSE instructions (movaps xmm0, ...) while the upper halves of the YMM registers hold data from VEX AVX code (vmulps ymm0, ...) costs extra unless a vzeroupper intervenes. On Sandy Bridge through Broadwell each switch triggers a state save or restore of dozens of cycles; on Skylake and later each legacy SSE instruction instead carries an extra blend μop and a dependency on the full register[32]. Compilers insert vzeroupper before calls and returns in AVX code, and intrinsic code compiled with AVX enabled is VEX-encoded throughout. Hand-written assembly that mixes the two has to insert vzeroupper itself.
    Forgetting the scalar tail. A loop processing 8 floats at a time over an array of 503 elements leaves 7 elements undone. Common bug patterns: a vector loop bound that reads past the end (i < count where i + 8 <= count was meant), a tail loop that starts at the wrong index and redoes or skips elements. AVX-512's masked load and store make the tail a non-issue; pre-AVX-512 code should pad arrays to a multiple of the SIMD width and zero the padding, or write the scalar tail with a clear loop bound.
    Treating scalar SSE math as cheaper than packed. addss and addps have identical latency and throughput on mainstream Intel and AMD cores of the last fifteen years[1]. Three lanes of an XMM register sitting unused doesn't make the instruction faster. The win is in using those lanes.
    Cross-lane shuffles on the critical path. vperm2f128, vpermps, and vextractf128 have 3-cycle latency on Skylake, against 1 cycle for in-lane shuffles[1]. A horizontal sum at the end of a long reduction amortizes this; a cross-lane shuffle inside a loop-carried dependency chain adds two cycles to every iteration.
    Gather on cold data. A gather's result isn't ready until its slowest lane arrives, so one cache miss among eight lanes stalls everything that depends on the whole register. On a large working set with random indices, a gather buys little over scalar loads. Restructure the data instead.
    Compiling for the dev machine's instruction set. A binary compiled with -march=skylake-avx512 can crash with SIGILL (the illegal-instruction signal) on a chip without AVX-512. Shipping game binaries either pick a conservative baseline (x86-64-v2, or x86-64-v3 for titles that require AVX2) or dispatch at startup. Test the dispatch path on hardware that lacks the higher-tier instructions; the failure mode is "works on the dev machine, crashes at boot on a player's machine"[27].
    Floating-point determinism. A reduction reordered for SIMD doesn't produce the same bit pattern as a scalar left-to-right reduction. For replays, lockstep multiplayer, and deterministic physics, write the SIMD reduction tree explicitly (the same tree every time, regardless of length) and don't enable -fassociative-math[15].
    Denormals. A particle that has decayed to a near-zero velocity can produce subnormal numbers (floats too close to zero for the normal format, stored with less precision), and on many Intel cores an operation that produces or consumes one can take a microcode assist costing on the order of a hundred cycles[32]. The fix is to set Flush-to-Zero (FTZ) and Denormals-Are-Zero (DAZ) in MXCSR, the SSE control register; physics and audio codebases routinely do. MXCSR is per thread, so set it on every worker thread, not just the main one.

Worked example

A particle's speed starts at 1.0 and is damped by 0.9 every frame. It drops below the smallest normal float, about 1.18 × 10−38, after 829 frames, about 14 s at 60 fps.

It never reaches zero: with round-to-nearest, 0.9 times a tiny subnormal rounds back to the same value, and the speed settles at about 5.6 × 10−45. Without FTZ and DAZ, every damping multiply on that particle from then on works on a subnormal.

[Back to SIMD from Scratch](index.md)
