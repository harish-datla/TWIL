# SIMD: SSE, AVX, AVX-512


A particle update adds each particle's velocity to its position, a few thousand times a frame. Doing those adds one float at a time leaves most of the CPU's arithmetic width unused. SIMD (Single Instruction, Multiple Data) replaces a loop body that operates on one scalar with one instruction that operates on a vector of values. Every x86-64 CPU has SSE2; nearly every mainstream Core, Ryzen, and current console chip from the last decade has AVX2; Intel server chips and AMD Zen 4 and later have AVX-512[10]. The mental model:

    An XMM register holds 128 bits = 4 floats = 2 doubles = 16 bytes = 4 int32s.
    A YMM register holds 256 bits = 8 floats = 4 doubles = 32 bytes = 8 int32s.
    A ZMM register holds 512 bits = 16 floats = 8 doubles = 64 bytes = 16 int32s.
    Each value in the register is a lane. SIMD instructions act on every lane in parallel: vaddps ymm0, ymm1, ymm2 adds eight pairs of single-precision floats simultaneously.

The suffix on a SIMD mnemonic names the lane format. ps = packed single (4/8/16 floats), pd = packed double, ss = scalar single (one float, others untouched), sd = scalar double. Integer variants use b/w/d/q for byte/word/dword/qword. vpaddd ymm0, ymm1, ymm2 is eight 32-bit integer adds in parallel[17].

Worked example

The widget below adds ymm1 = (6, 1, 5, 9, 9, 3, 6, 7) and ymm2 = (7, 3, 3, 5, 8, 6, 5, 4) with one vaddps: ymm0 = (13, 4, 8, 14, 17, 9, 11, 11).

3,000 particles with x, y and z positions is 9,000 float adds: 9,000 scalar addss, or 9,000 / 8 = 1,125 vaddps.

The widget below shows one SIMD operation across a register. vaddps on a YMM register adds two eight-lane vectors in a single instruction. A highlight sweeps across the lanes so each lane index is easy to follow; the hardware computes all eight in the same operation:
Live · SIMD lanes in flight
vaddps (8× add)
vfmadd (8× a*b+c)
vshufps (rearrange lanes)
The top rows are the source registers and the bottom row the destination, one cell per lane; switch between the add, the fused multiply-add and the shuffle. A YMM add or FMA is one μop on Skylake and on Zen 2 and later (Zen 1 splits it into two 128-bit halves), and Skylake can start two of them per cycle; the result is ready 4 cycles later[13]. The sweep is slowed down for readability. In the vshufps view, immediate 0xEE takes elements 2 and 3 of the first source, then 2 and 3 of the second, separately in each 128-bit half.


[Back to x86-64 Assembly from Scratch](index.md)
