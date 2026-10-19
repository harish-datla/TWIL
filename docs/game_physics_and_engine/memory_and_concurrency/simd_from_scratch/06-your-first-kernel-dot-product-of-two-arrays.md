# Your First Kernel: Dot Product of Two Arrays


A lighting pass, a physics solver and a neural-network layer all end up computing the dot product of two long float arrays: multiply matching entries and add the products up. It's a reduction, a loop that folds a whole array into one value, and it's the smallest kernel that shows both the lanes and the dependency chains at work. The scalar form is one multiply-add per element:
dot_scalar.cpp

float dotScalar(const float* aArray, const float* bArray, int elementCount) {
  float accumulator = 0.0f;
  for (int i = 0; i < elementCount; ++i) {
    accumulator += aArray[i] * bArray[i];
  }
  return accumulator;
}

The natural AVX2 form processes eight elements per iteration, accumulating into a single 256-bit vector and reducing to a scalar at the end:
dot_avx2_naive.cpp · one accumulator

float dotAvx2Naive(const float* aArray, const float* bArray, int elementCount) {
  __m256 accumulator = _mm256_setzero_ps();
  int i = 0;
  for (; i + 8 <= elementCount; i += 8) {
    __m256 aVector = _mm256_loadu_ps(aArray + i);
    __m256 bVector = _mm256_loadu_ps(bArray + i);
    // Fused multiply-add: accumulator += aVector * bVector, one rounding step.
    accumulator = _mm256_fmadd_ps(aVector, bVector, accumulator);
  }
  // Horizontal sum of the 8 lanes in `accumulator`. Reduce-and-shuffle tree.
  __m128 lowHalf  = _mm256_castps256_ps128(accumulator);            // lanes 0..3
  __m128 highHalf = _mm256_extractf128_ps(accumulator, 1);          // lanes 4..7
  __m128 sum128 = _mm_add_ps(lowHalf, highHalf);                    // 4 sums
  sum128 = _mm_hadd_ps(sum128, sum128);                             // 2 sums in low two lanes
  sum128 = _mm_hadd_ps(sum128, sum128);                             // 1 sum in low lane
  float totalSum = _mm_cvtss_f32(sum128);

  // Scalar tail for the last 0..7 elements that didn't fit a vector iteration.
  for (; i < elementCount; ++i) totalSum += aArray[i] * bArray[i];
  return totalSum;
}

Three working parts here, each of which is a SIMD pattern you'll reuse:

    The main loop. One FMA per iteration. An FMA can take only one of its inputs straight from memory, so the compiler emits one plain load and folds the other into the FMA's memory operand: two loads plus one FMA per eight elements[1].
    The horizontal sum. Reducing an 8-lane vector to one scalar: add the two 128-bit halves, then fold the remaining four lanes. vhaddps is the SSE3 horizontal-add instruction; it's used twice to fold four lanes into one.
    The scalar tail. Loop iterations that don't fit a full vector. For elements past the last i + 8 <= elementCount boundary, fall back to scalar. AVX-512 has an alternative using a masked load (§8) that handles the tail without a separate loop.

On Skylake this kernel runs roughly one iteration every 4 cycles in steady state, capped by the loop-carried dependency on accumulator. vfmadd231ps has a 4-cycle latency on Skylake; each iteration's FMA depends on the previous, so the loop is latency-bound, not throughput-bound[1]. That is the serial-chain problem from §1: the FMA ports sit idle most of the time while the single dependency chain serializes.

Worked example

A dot product of 1,024 floats on Skylake. The vector loop runs 1,024 / 8 = 128 iterations, each waiting 4 cycles for the previous FMA: 512 cycles, plus roughly 15 to 20 for the horizontal sum.

The scalar loop's chain also waits 4 cycles per step, but each step handles one element: 1,024 × 4 = 4,096 cycles. The vector loop is 8× faster and still fills only one of the 8 FMA slots (2 ports × 4 cycles) each iteration offers.

The fix is in §7: use multiple accumulators to break the chain.


[Back to SIMD from Scratch](index.md)
