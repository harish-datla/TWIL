# Breaking the Dependency Chain


The §6 kernel uses eight lanes but leaves seven of every eight FMA slots empty, because every iteration's FMA depends on the previous iteration's accumulator. The fix is to keep several independent accumulators and sum them at the end. How many you need is set by Skylake's FMA pipeline: 4-cycle latency, two-per-cycle throughput. Little's law (items in flight = start rate × time each one takes) gives the number of independent FMAs that have to be in flight at once to keep both ports busy: latency × throughput = 4 × 2 = 8. With eight independent accumulator chains, both FMA ports can start a new FMA every cycle, as long as nothing else runs out first[1].
dot_avx2_unrolled.cpp · eight independent accumulators

float dotAvx2Unrolled(const float* aArray, const float* bArray, int elementCount) {
  // Eight independent partial sums. Each is its own dependency chain;
  // with FMA latency 4 and throughput 2, eight in-flight FMAs saturate the ports.
  __m256 sum0 = _mm256_setzero_ps(), sum1 = _mm256_setzero_ps();
  __m256 sum2 = _mm256_setzero_ps(), sum3 = _mm256_setzero_ps();
  __m256 sum4 = _mm256_setzero_ps(), sum5 = _mm256_setzero_ps();
  __m256 sum6 = _mm256_setzero_ps(), sum7 = _mm256_setzero_ps();

  int i = 0;
  for (; i + 64 <= elementCount; i += 64) {
    // 8 independent FMAs per iteration; the CPU schedules across both FMA ports.
    sum0 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i +  0), _mm256_loadu_ps(bArray + i +  0), sum0);
    sum1 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i +  8), _mm256_loadu_ps(bArray + i +  8), sum1);
    sum2 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i + 16), _mm256_loadu_ps(bArray + i + 16), sum2);
    sum3 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i + 24), _mm256_loadu_ps(bArray + i + 24), sum3);
    sum4 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i + 32), _mm256_loadu_ps(bArray + i + 32), sum4);
    sum5 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i + 40), _mm256_loadu_ps(bArray + i + 40), sum5);
    sum6 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i + 48), _mm256_loadu_ps(bArray + i + 48), sum6);
    sum7 = _mm256_fmadd_ps(_mm256_loadu_ps(aArray + i + 56), _mm256_loadu_ps(bArray + i + 56), sum7);
  }
  // Fold the eight accumulators together in pairs, then a horizontal sum.
  __m256 fold01 = _mm256_add_ps(sum0, sum1);
  __m256 fold23 = _mm256_add_ps(sum2, sum3);
  __m256 fold45 = _mm256_add_ps(sum4, sum5);
  __m256 fold67 = _mm256_add_ps(sum6, sum7);
  __m256 fold0123 = _mm256_add_ps(fold01, fold23);
  __m256 fold4567 = _mm256_add_ps(fold45, fold67);
  __m256 totalVector = _mm256_add_ps(fold0123, fold4567);

  __m128 sum128 = _mm_add_ps(_mm256_castps256_ps128(totalVector),
                             _mm256_extractf128_ps(totalVector, 1));
  sum128 = _mm_hadd_ps(sum128, sum128);
  sum128 = _mm_hadd_ps(sum128, sum128);
  float totalSum = _mm_cvtss_f32(sum128);
  for (; i < elementCount; ++i) totalSum += aArray[i] * bArray[i];
  return totalSum;
}

Each iteration now issues eight independent FMAs, and the latency chain no longer limits the loop. The FMA ports alone could now sustain two FMAs per cycle, 8× the single-accumulator loop's 0.25. This dot product hits a different wall first: every FMA consumes two fresh 32-byte loads, and Skylake has two load ports, so the loop tops out at one FMA per cycle, 8 elements per cycle, or 4× the single-accumulator version[1]. Four accumulators already reach that limit (4 chains ÷ 4-cycle latency = one FMA per cycle), which is one reason many production dot products stop at four.

Eight accumulators pay off when an FMA needs at most one load, so the FMA ports become the limit: a sum of squares, a polynomial evaluated across many inputs, the matrix transform in §11 where one operand is a broadcast held in a register. They also help on cores with more load bandwidth, such as Golden Cove with three load ports. Sixteen accumulators rarely help on AVX2 hardware because there are only sixteen YMM registers and the extra accumulators start to spill. Ice Lake / Sunny Cove and AMD Zen 3 and Zen 4 have the same 4-cycle FMA latency and two-per-cycle throughput, so the same arithmetic applies[1][6].

Worked example

The same 1,024 floats, 128 FMAs. One accumulator: 128 × 4 = 512 cycles.

Eight accumulators, counting only the FMA ports: two FMAs start per cycle, the last at cycle 63, and its result is ready at cycle 67, about 7.6× faster. With Skylake's two load ports feeding two loads to every FMA, only one FMA starts per cycle: about 128 cycles, 4× faster.

The widget models just the two FMA ports for a 32-FMA trace (256 elements) under both accumulator schemes, so it shows the latency effect in isolation:
Live · Reduction race (single vs eight accumulators)
Single accumulator
Eight accumulators
Run ▶
Reset ⟲
cycles (modelled)
128
FMAs per cycle
0.25
speedup vs single
1.0×
Each bar is one FMA on port 0 or port 1, drawn from the cycle it starts to the cycle its result is ready. Run the single accumulator, then eight accumulators, and compare the cycle readouts: 128 against 19. A simplified Skylake model of the FMA ports only: vfmadd231ps ymm has 4-cycle latency and two-per-cycle throughput on ports 0 and 1[1]. The single-accumulator loop is latency-bound at one FMA per 4 cycles (0.25 FMA/cycle). With eight accumulators the ports reach their ceiling of two FMAs per cycle. The readout shows 6.7× rather than 8× because the trace is only 32 FMAs long and the last FMAs still take their full 4 cycles to finish; on a long array that fixed cost amortizes. Load ports aren't modelled: for the dot product they cap Skylake at one FMA per cycle, as the text above explains.

-O3 -ffast-math (or specifically -fassociative-math, which GCC honors only together with -fno-signed-zeros and -fno-trapping-math) is what lets the autovectorizer perform this transformation for you. Without it, floating-point addition's non-associativity prevents the compiler from regrouping the sum[15]. The compiler's autovectorization report (-fopt-info-vec on GCC, -Rpass=loop-vectorize on Clang) tells you when it succeeded.

A caveat: the reordered sum produces a slightly different result than the strict left-to-right sum. The two usually differ in the last few ULPs (units in the last place: the gap between adjacent floats at that size), more when large terms cancel[15]; neither is more correct in general, but they're not bit-identical. If your engine needs deterministic, cross-platform reproducible results (replays, multiplayer lockstep, deterministic physics), the reordered sum can be a problem. The fix is to fix the reduction tree shape (always the same tree, regardless of array length) and never let the compiler re-associate, which means leaving -fassociative-math off and writing the SIMD form explicitly.

Worked example

Sum 100,000,000 + 1 − 100,000,000 + 1 in float. Left to right: floats near 108 are 8 apart, so 108 + 1 rounds back to 108; subtracting 108 leaves 0, and the last 1 gives 1.

Two accumulators, the way a vectorized loop splits the work: the even elements give 108 − 108 = 0, the odd ones 1 + 1 = 2, total 2. The exact answer is 2, so here the reordered sum wins; with other inputs the left-to-right one does. Either way the bits differ, and a replay recorded with one won't reproduce with the other.

[Back to SIMD from Scratch](index.md)
