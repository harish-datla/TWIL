# SIMD from Scratch

A frustum cull tests thousands of bounding spheres against the same six planes every frame, and a particle update runs the same multiply-add on every particle. Written as an ordinary loop, each instruction handles one value. Every x86-64 CPU has vector registers that hold four floats, most from the last decade hold eight, and some hold sixteen; one SIMD (single instruction, multiple data) instruction does the same operation on all of them at once. The speedup often comes less from a better algorithm than from laying the data out so the vector unit can consume it lane by lane. This page shows how to lay data out for it, how to write kernels with intrinsics (C++ functions that each compile to one vector instruction), and when the compiler does it for you. It works up from the lane model to an 8-at-a-time frustum cull and a batched 4×4 matrix transform, on x86's SSE, AVX and AVX-512 (its 128-, 256- and 512-bit instruction sets) and Arm's 128-bit NEON, with claims cited to vendor manuals and published measurements.
Time
~75 min
Level
Engine programmer, intermediate to senior
Prereqs
You can read C or C++ comfortably and know what a cache line is. The Assembly tutorial pairs naturally.
Hardware
Any x86-64 chip from the last decade, or an Apple Silicon / Switch / Android device for the ARM sections
◂ Build a Game Engine
Phase 1 · Memory & concurrency
Next · Lock-free Queues ▸
What you'll be able to do by the end

    Read SIMD intrinsics and the disassembly they produce across SSE, AVX2, FMA and AVX-512, and tell from a mnemonic how many lanes it works on.
    Pick AoS, SoA or AoSoA for a system from its access pattern, and work out how many bytes a pass pulls through the cache in each.
    Write a reduction that keeps both FMA units busy, and replace per-element branches, loop tails and small lookups with masks, blends and shuffles.
    Transform 8 vertices and cull 8 bounding spheres per instruction, and estimate what each costs in cycles.
    Decide when to trust the autovectorizer, when AVX-512 is worth shipping, and how to port the same math to Arm NEON.


## Sections

- [What SIMD Buys You, and What It Costs](01-what-simd-buys-you-and-what-it-costs.md)
- [A Short History of x86 Vector Extensions](02-a-short-history-of-x86-vector-extensions.md)
- [The Lane Model: How a SIMD Register Actually Works](03-the-lane-model-how-a-simd-register-actually-works.md)
- [AoS, SoA, and AoSoA: The Layout Decision That Dominates Everything](04-aos-soa-and-aosoa-the-layout-decision-that-dominates-everything.md)
- [Alignment, Loads, and the Cache-Line Crossing](05-alignment-loads-and-the-cache-line-crossing.md)
- [Your First Kernel: Dot Product of Two Arrays](06-your-first-kernel-dot-product-of-two-arrays.md)
- [Breaking the Dependency Chain](07-breaking-the-dependency-chain.md)
- [Masking and Predication: Conditionals without Branches](08-masking-and-predication-conditionals-without-branches.md)
- [Shuffles, Blends, Broadcasts: Rearranging the Lanes](09-shuffles-blends-broadcasts-rearranging-the-lanes.md)
- [Gather and Scatter: When You Have to Load from N Pointers](10-gather-and-scatter-when-you-have-to-load-from-n-pointers.md)
- [A Worked Engine Kernel: Batched 4x4 Matrix-Vector Transform](11-a-worked-engine-kernel-batched-4x4-matrix-vector-transform.md)
- [A Worked Engine Kernel: 8 Spheres against the Frustum per Iteration](12-a-worked-engine-kernel-8-spheres-against-the-frustum-per-iteration.md)
- [Autovectorization, Intrinsics, and When Each Wins](13-autovectorization-intrinsics-and-when-each-wins.md)
- [AVX-512 in 2026: The Downclocking Story](14-avx-512-in-2026-the-downclocking-story.md)
- [ARM: NEON, SVE2, and Porting the Math](15-arm-neon-sve2-and-porting-the-math.md)
- [Pitfalls](16-pitfalls.md)
- [What's Next](17-whats-next.md)
- [Sources & Further Reading](18-sources-and-further-reading.md)
