# The Lane Model: How a SIMD Register Actually Works


Open a hot loop in a profiler's disassembly view and you'll see lines like vaddps ymm0, ymm1, ymm2 or vaddss xmm0, xmm0, xmm1. Reading them takes one idea, the lane. A SIMD register holds several values of the same type, in fixed positions called lanes. One instruction operates on every lane at the same time. The instruction's mnemonic encodes the lane element format: ps for packed single-precision floats, pd for packed doubles, b/w/d/q for packed 8 / 16 / 32 / 64-bit integers. The scalar variants are ss (scalar single) and sd (scalar double); they touch only the low lane and leave the rest unchanged[9].

The same physical register file appears at three widths, the 128-bit XMM registers, the 256-bit YMM registers and the 512-bit ZMM registers, and each width is a strict prefix of the next:
Register	Width	Lanes (float)	Lanes (double)	Lanes (int32)	Lanes (byte)
xmm0–xmm15	128 bits	4	2	4	16
ymm0–ymm15	256 bits	8	4	8	32
zmm0–zmm31	512 bits	16	8	16	64

The lower 128 bits of ymm0 is xmm0. The lower 256 bits of zmm0 is ymm0. AVX-512-capable cores extend the architectural register count to thirty-two (xmm0–xmm31, etc., reachable only through AVX-512's EVEX encoding); pre-AVX-512 cores see sixteen[2].

Worked example

A YMM register is 256 bits. A float is 32 bits, so it holds 256 / 32 = 8 floats; a double is 64 bits, so 4; a byte is 8 bits, so 32.

The mnemonic says which: vaddps ymm adds 8 floats, vaddpd ymm adds 4 doubles, vpaddb ymm adds 32 bytes. Same register, same one instruction.

The widget below shows one instruction at the three widths. vaddps on a YMM register adds eight pairs of single-precision floats in one μop (micro-operation, the unit the core schedules and counts); on a ZMM register it adds sixteen. Every lane pulses at once because the hardware computes all of them in the same operation:
Live · SIMD lanes in flight
vaddps (add)
vfmadd231ps (a += b·c)
vmulps (multiply)
xmm (4 lanes)
ymm (8 lanes)
zmm (16 lanes)
Each column is one lane: the source registers on top, the result in the bottom row. Switch the operation and the width; the whole row lights up at once at every width. In the FMA view, lane 0 computes 1 + 6 × 8 = 49 and lane 1 computes 7 + 1 × 5 = 12, both in the same instruction. A 256-bit add or FMA is one μop, and Skylake, Ice Lake, and Zen 2 and later can start two per cycle. The result is ready 2 to 4 cycles after the instruction starts, depending on the operation and the core[1]; the pulse here is slowed down so it is visible.

Two facts about the lane model that beginners get wrong, and that the rest of this tutorial depends on:

    A scalar op costs the same as a packed one. vaddss xmm0, xmm1, xmm2 adds the low single-precision floats of xmm1 and xmm2; the upper three lanes of the result are copied from xmm1 (the VEX form, AVX's encoding with a separate destination register, copies them from the first source; legacy SSE addss leaves the destination's upper lanes in place). On Skylake a scalar addss has identical latency and throughput to a packed addps: both are one μop on the same two ports[1]. Scalar SSE math is not cheaper than packed SSE math; it does one lane of work for the same cost.
    Lane index is positional, not by name. A 256-bit YMM register is not a struct with named fields. Whether lane 0 holds x, lane 1 holds y, etc. is your convention. Most SoA code dedicates a whole register to one component (a register full of x values from many particles), not to one record with four components.

The intrinsic-level vocabulary, since the compiler hides the register names behind type-tagged values[9]:
simd_types.cpp · the intrinsic types and their loads

#include <immintrin.h>   // pulls in every Intel intrinsic header

// SSE / SSE2: 128-bit registers. Type tag carries the element format.
__m128   vec4Floats;   // 4 × float
__m128d  vec2Doubles;  // 2 × double
__m128i  vec16Bytes;   // 16 × int8, or 8 × int16, or 4 × int32, or 2 × int64

// AVX / AVX2: 256-bit registers.
__m256   vec8Floats;
__m256d  vec4Doubles;
__m256i  vec32Bytes;

// AVX-512: 512-bit registers.
__m512   vec16Floats;
__m512d  vec8Doubles;
__m512i  vec64Bytes;

// Mask register (AVX-512). One bit per lane; eight for __m512d, sixteen for __m512.
__mmask8  doubleMask;
__mmask16 floatMask;

// Load from memory. _ps = packed single, _pd = packed double, _si* = integer.
float* sourcePointer = ...;
__m256 loaded = _mm256_loadu_ps(sourcePointer);   // unaligned load: any address
__m256 loadedAligned = _mm256_load_ps(sourcePointer);  // requires 32-byte aligned source
_mm256_storeu_ps(destinationPointer, loaded);     // unaligned store

Aligned and unaligned loads have identical performance on every Intel core from Nehalem (2008) onward when the address actually is aligned; the cost only appears when an unaligned load crosses a cache-line boundary, which adds a few cycles[10]. Modern code uses _mm256_loadu_ps by default and accepts the rare cache-line crossing as the cost of not having to manually align every buffer.
Check Your Understanding Q1 of 1

You compile a tight loop that does output[i] = a[i] + b[i] over 2,000 floats with gcc -O3 -march=skylake, and call it repeatedly so all three arrays stay in the L1 cache. Profiling shows the loop runs at roughly 8 floats per cycle. Which of these is the most plausible explanation?
The compiler autovectorized the loop to 256-bit YMM adds, and the loop is now load/store bound: each iteration needs two 256-bit loads and one 256-bit store, and Skylake's single store port caps it at one iteration, 8 floats, per cycle.
The loop is still scalar, but Skylake's out-of-order window keeps eight independent addss μops in flight at once, and retirement is wide enough to sustain eight scalar adds, 8 floats, per cycle with no SIMD instructions in the loop at all.
The compiler unrolled the loop eight ways but kept it scalar; eight separate accumulator chains hide the 4-cycle FP-add latency, so the two FP-add ports retire eight independent scalar adds every cycle between them.
0/1 correct

[Back to SIMD from Scratch](index.md)
