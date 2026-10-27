# μops, Ports, and Dependency Chains in Practice


Summing a float array is about the simplest loop there is, and the obvious version uses a sixty-fourth of what the core can do. §3 sketched the out-of-order engine, which runs instructions as soon as their inputs are ready rather than in program order. With the SIMD walkthrough in mind, here is what microarchitectural tuning looks like in practice. Take a reduction: summing an array of floats that fits in L1.
sum_serial.cpp · the naive loop

float sumSerial(const float* values, int count) {
  float sum = 0;
  for (int i = 0; i < count; ++i) sum += values[i];   // each add waits for the previous one
  return sum;
}

Compiled with -O2 (no fast-math), the inner loop is one addss per element. addss on Skylake has a latency of 4 cycles and a throughput of 2 per cycle[13]. Because each iteration depends on sum from the previous iteration, the loop is bound by the latency, not the throughput: one element every 4 cycles. The second add port and most of the other ports sit idle.

The fix is to break the dependency chain: use several accumulators that the CPU can work on independently. -O3 -ffast-math on a modern compiler does this for you (-fassociative-math is the specific permission needed), producing something like:
sum_parallel.cpp · multi-accumulator + SIMD, after autovectorization

float sumParallel(const float* values, int count) {
  // Four independent vector accumulators, eight floats each. Assumes count is a
  // multiple of 32; the compiler's version adds a tail loop.
  __m256 sum0 = _mm256_setzero_ps(), sum1 = _mm256_setzero_ps();
  __m256 sum2 = _mm256_setzero_ps(), sum3 = _mm256_setzero_ps();
  for (int i = 0; i < count; i += 32) {
    sum0 = _mm256_add_ps(sum0, _mm256_loadu_ps(values + i + 0));
    sum1 = _mm256_add_ps(sum1, _mm256_loadu_ps(values + i + 8));
    sum2 = _mm256_add_ps(sum2, _mm256_loadu_ps(values + i + 16));
    sum3 = _mm256_add_ps(sum3, _mm256_loadu_ps(values + i + 24));
  }
  __m256 total = _mm256_add_ps(_mm256_add_ps(sum0, sum1), _mm256_add_ps(sum2, sum3));
  // horizontal sum of total into a scalar; details elided
  ...
}

Four independent accumulators, each a 256-bit vector of eight floats. Each accumulator is still a chain with 4-cycle latency, so each one takes a new add every 4 cycles, and four chains give one 256-bit add per cycle: 8 floats per cycle. Compared to the naive loop's 0.25 floats per cycle, that is a 32× speedup, 8× from the SIMD width and 4× from running four chains at once. It is still half of what Skylake can do. Skylake has two 256-bit add ports and two 256-bit load ports[13][12], so eight accumulators (latency 4 × two adds per cycle, the same arithmetic as the SIMD tutorial's reduction chapter) would reach 16 floats per cycle, where the add ports and the load ports both saturate. That peak holds only while the data is in L1. For a larger array the loop becomes bound by L2, L3, or DRAM bandwidth instead: a million floats is 4 MB, far more than Skylake's L2.

Worked example

Sum 1,024 floats on Skylake.

Naive loop: one dependent addss per float at 4 cycles each, 1,024 × 4 = 4,096 cycles.

Four 8-wide accumulators: 1,024 / 8 = 128 vector adds, 32 per accumulator, and the four chains run side by side, so 32 × 4 = 128 cycles.

Eight accumulators: 16 adds per chain, two chains starting each cycle, about 64 cycles. That's the 64× between the obvious loop and the core's peak.

Three tools check this kind of analysis. llvm-mca[27] takes a chunk of assembly and reports the expected steady-state IPC and per-port pressure. uica.uops.info[28] predicts the same from a more detailed simulation of Intel cores. perf stat -e cycles,instructions,branches,branch-misses,... on Linux measures what actually happened, from the CPU's performance counters.

[Back to x86-64 Assembly from Scratch](index.md)
