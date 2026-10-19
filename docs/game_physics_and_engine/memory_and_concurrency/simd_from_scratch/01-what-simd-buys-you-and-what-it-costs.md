# What SIMD Buys You, and What It Costs


A particle update, a cull pass and a skinning pass each run the same few lines of math on thousands of elements every frame, and a plain loop feeds the CPU one element per instruction. A modern desktop CPU core can issue two 256-bit floating-point multiply-adds (FMAs, which compute a × b + c in one instruction) per cycle, eight floats in each, from a single hardware thread[1]. The same core, running the same kernel one scalar element at a time, peaks at one-eighth of that: the two FMA ports (the issue slots in front of the FMA units) issue at the same rate either way, but each scalar FMA produces one lane instead of eight.

Worked example

A core at 4 GHz with two FMA ports. Vector code: 2 FMAs × 8 lanes × 4 billion cycles per second = 64 billion FMAs per second. Scalar code: 2 × 1 × 4 billion = 8 billion.

Real kernels land below both ceilings, but the ratio between them, 8, is the most 256-bit SIMD can buy on floats.

In a shipping engine the gap usually shows up as headroom rather than as a peak throughput number. A scalar particle tick can fall to a fraction of its cost once the data is in SoA form and the integration is one vfmadd231ps per axis per eight particles; the same holds for skinning (bending a character's mesh to follow its skeleton), instance-transform passes, frustum and occlusion culling, and the math-heavy parts of physics narrow-phase. The realized speedup varies with the kernel and the layout. The vector form processes 8 or 16 records per instruction while the loop overhead stays roughly fixed, and the time saved goes to other systems or more entities.

SIMD only pays off when:

    Your data is laid out for it. A vector load picks up sixteen contiguous bytes (SSE) or thirty-two (AVX); it does not pick up four scattered x components from four scattered Particle records. Whether the inputs are in SoA form (structure of arrays: one array per field) or AoS form (array of structures: one struct per element) usually decides whether a SIMD rewrite is a win. §4 is the long answer.
    The work has lane-level parallelism. A floating-point sum where each iteration adds into the previous result is one serial chain, bound by the add's latency, the cycles before one add's result can feed the next. Vectorizing it means regrouping the sum into independent partial sums, which the compiler won't do under strict IEEE-754 rules. §7.
    Branches in the inner loop are rare or vectorizable. A vector can't take a branch in some lanes and not others, so the SIMD form computes both sides of the branch and blends the results. AVX-512's mask registers are the first x86 SIMD with first-class predication: running an instruction only in the lanes where a condition holds. §8.
    The compiler did not already do it for you. Modern autovectorizers, the compiler passes that turn scalar loops into SIMD loops on their own, handle clean array maps on contiguous data, and floating-point reductions when allowed to reorder them. Hand-rolled intrinsics are the right answer when the compiler bailed out on the pattern, not when it didn't. §13.

The two engine kernels this page builds, a batched 4×4 matrix transform and an 8-at-a-time frustum cull, are each under 50 lines of AVX2 intrinsics. Six in-browser widgets step through the ideas on the way: the lanes, the layouts, a reduction, masking, a byte lookup and the cull itself.

The ceiling on the speedup is the lane count: for float data, 8× on AVX/AVX2 and 16× on AVX-512. Real kernels land below it, often well below, because layout conversion, tail handling, shuffles, and memory bandwidth each take a share. The rest of this tutorial makes those costs visible.

[Back to SIMD from Scratch](index.md)
