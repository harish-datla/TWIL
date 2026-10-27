# ARM: NEON, SVE2, and Porting the Math


The same engine ships on the Switch, on phones and on Apple Silicon Macs, and none of them run SSE or AVX. Arm's AArch64, the 64-bit Arm architecture (Armv8-A and later), makes NEON its mandatory SIMD: thirty-two 128-bit registers, V0–V31. NEON is roughly comparable to SSE4 (128-bit, no native masking, no gather). For game programmers it's the Nintendo Switch, Switch 2, Apple Silicon Macs, current iPhones and Android phones, and the AWS Graviton server line[28].

The translation from x86 SSE/AVX2 intrinsics to NEON is mostly mechanical. _mm_add_ps ↔ vaddq_f32. _mm_mul_ps ↔ vmulq_f32. _mm_fmadd_ps(a, b, c) ↔ vfmaq_f32(c, a, b): note that NEON takes the accumulator first. _mm_loadu_ps ↔ vld1q_f32. The dot-product kernel from §6 with NEON intrinsics:

Worked example

With a = 2, b = 3, c = 1 in every lane, _mm_fmadd_ps(a, b, c) and vfmaq_f32(c, a, b) both compute 2 × 3 + 1 = 7.

Port it by copying the argument order, vfmaq_f32(a, b, c), and NEON computes a + b × c = 2 + 3 × 1 = 5. It compiles, runs and is wrong.
dot_neon.cpp · the same kernel, NEON

#include <arm_neon.h>

float dotNeon(const float* aArray, const float* bArray, int elementCount) {
  float32x4_t accumulatorA = vdupq_n_f32(0.0f);    // 4-lane register of zeros
  float32x4_t accumulatorB = vdupq_n_f32(0.0f);
  int i = 0;
  for (; i + 8 <= elementCount; i += 8) {
    float32x4_t aLow  = vld1q_f32(aArray + i);
    float32x4_t bLow  = vld1q_f32(bArray + i);
    float32x4_t aHigh = vld1q_f32(aArray + i + 4);
    float32x4_t bHigh = vld1q_f32(bArray + i + 4);
    accumulatorA = vfmaq_f32(accumulatorA, aLow,  bLow);    // fma: dst + a*b, one rounding step
    accumulatorB = vfmaq_f32(accumulatorB, aHigh, bHigh);
  }
  float32x4_t combined = vaddq_f32(accumulatorA, accumulatorB);
  // Horizontal sum: AArch64 has vaddvq_f32, one intrinsic that reduce-adds all 4 lanes.
  float totalSum = vaddvq_f32(combined);
  for (; i < elementCount; ++i) totalSum += aArray[i] * bArray[i];
  return totalSum;
}

Two NEON conveniences SSE doesn't have. vaddvq_f32 is a single intrinsic for the horizontal sum (it compiles to two faddp pairwise adds; integer reductions such as addv are single instructions). And NEON arithmetic has always used three-operand forms, so there's no legacy-SSE-style "destination is one of the sources" constraint. Apple's CPU optimization guide for the M-series and Arm's per-core software optimization guides are the practical references[29][30].
SVE and SVE2: vector-length-agnostic

The longer-term ARM story is SVE (Scalable Vector Extension), which adds vector-length-agnostic instructions: the same binary runs on hardware whose vectors are any multiple of 128 bits, up to 2048. SVE2 is an optional feature in the ARMv9-A profile (not mandatory), implemented in practice by the Neoverse server line and a slice of mobile silicon[31]. The shipping examples worth knowing about:

    Fujitsu A64FX in the Fugaku supercomputer. SVE (not SVE2) at 512 bits per vector. First major SVE deployment, 2020.
    AWS Graviton 3 (Neoverse V1). SVE at 256 bits, implemented as two 256-bit pipes. The first cloud-server SVE rollout, 2022.
    AWS Graviton 4 (Neoverse V2). SVE2 at 128 bits, four 128-bit pipes. Arm narrowed the vector width from V1 to V2; peak FP throughput per core is unchanged because V2's four 128-bit pipes match V1's two 256-bit ones.
    Armv9 client parts. Phone SoCs based on Cortex-X4 / X925 / A720 implement SVE2 at 128 bits. Apple's M4 (Armv9.2-A, 2024) implements SME (Scalable Matrix Extension) but does not expose non-streaming SVE2 to user code; SVE-style instructions on M4 are only available inside an SME streaming-mode region[34]. Code that wants to run on M4 has to use NEON or stay inside SME.

SVE has first-class predication: every instruction takes a predicate register, the way AVX-512 does, so the mask-and-blend dance from §8 disappears. Code written for SVE looks closer to AVX-512 with mask registers than to NEON. The price is that you can't reason about lane count at compile time; every loop has to be written in a "while there's still work" form (the SVE WHILELT instruction generates the predicate for "lanes within bounds").

Worked example

One SVE loop over 503 floats. On a core with 128-bit vectors (4 floats) it runs 126 iterations, the last with 3 active lanes. On the 512-bit A64FX (16 floats) the same binary runs 32 iterations, the last with 7 active lanes. WHILELT computes each last predicate; the code never names the width.

For game code in 2026 the practical situation is: NEON is what every shipping Arm target supports, including Switch 2 and every consumer Apple Silicon Mac. SVE2 is a server-side story (Graviton, other Neoverse-based instances) plus a slice of Android phones. The Highway library[22] has SVE and NEON backends, so kernels written against it can target both without a rewrite. Writing pure NEON intrinsics in 2026 is much like writing pure SSE intrinsics: it runs on everything in its family today but leaves the wider vector units on newer hardware unused.

[Back to SIMD from Scratch](index.md)
