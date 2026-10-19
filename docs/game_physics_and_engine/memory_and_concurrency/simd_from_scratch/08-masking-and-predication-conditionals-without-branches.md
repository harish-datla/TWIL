# Masking and Predication: Conditionals without Branches


A particle update has an if in it: dead particles don't move, live ones do. In scalar code that's a branch per particle, but eight lanes in one register can't each take a different branch. A pre-AVX-512 vector ALU has no conditional execution. vaddps ymm0, ymm1, ymm2 operates on every lane; there is no "operate on lanes where some condition holds." When the source has a branch whose outcome differs per lane, the SIMD form does the work for both sides and blends the result, with the blend driven by a comparison-produced mask. The simplest case needs no blend at all:
clamp_avx2.cpp · clamp(x, 0, 1) eight at a time, no branches

// Clamp every element of `inputArray` to [0, 1]. Written with if-statements, the scalar
// version is two compares per element; vminps / vmaxps do both tests without a branch.
// Assumes elementCount is a multiple of 8.
void clampZeroOne(float* inputArray, int elementCount) {
  __m256 zeroVector = _mm256_setzero_ps();
  __m256 oneVector  = _mm256_set1_ps(1.0f);
  for (int i = 0; i < elementCount; i += 8) {
    __m256 inputVector  = _mm256_loadu_ps(inputArray + i);
    // min(input, 1) then max(result, 0). Two SIMD min/max ops, no branches.
    __m256 belowOne  = _mm256_min_ps(inputVector, oneVector);
    __m256 inRange   = _mm256_max_ps(belowOne, zeroVector);
    _mm256_storeu_ps(inputArray + i, inRange);
  }
}

vminps and vmaxps are the cleanest case of vectorized branching: there's a hardware instruction for the exact lane-wise min and max, so no comparison-and-blend is needed. The general case (a predicate that isn't a comparison the SIMD ISA has a direct op for) uses a comparison followed by a blend:
condition_avx2.cpp · branchless lerp-or-skip

// Apply an alpha-blended override to every element where `maskArray[i]` is non-zero,
// keep the original value otherwise. In scalar, this is one branch per element.
// Assumes elementCount is a multiple of 8.
void conditionalLerp(float* destination, const float* overrideValue,
                     const int* maskArray, int elementCount, float alpha) {
  __m256 alphaVector   = _mm256_set1_ps(alpha);
  __m256 oneMinusAlpha = _mm256_set1_ps(1.0f - alpha);
  for (int i = 0; i < elementCount; i += 8) {
    __m256i predicateBits = _mm256_loadu_si256((const __m256i*)(maskArray + i));
    // All-ones in the lanes where maskArray[i] == 0. AVX2 has no integer "not equal"
    // compare, so test for zero and swap the blend operands below instead.
    __m256 maskIsZero = _mm256_castsi256_ps(
      _mm256_cmpeq_epi32(predicateBits, _mm256_setzero_si256()));

    __m256 originalLanes = _mm256_loadu_ps(destination + i);
    __m256 overrideLanes = _mm256_loadu_ps(overrideValue + i);
    __m256 blendedLanes  = _mm256_fmadd_ps(originalLanes, oneMinusAlpha,
                                           _mm256_mul_ps(overrideLanes, alphaVector));
    // blendv takes its second operand where the mask lane's sign bit is set:
    // original where maskArray[i] == 0, blended everywhere else.
    __m256 selectedLanes = _mm256_blendv_ps(blendedLanes, originalLanes, maskIsZero);
    _mm256_storeu_ps(destination + i, selectedLanes);
  }
}

Two SIMD primitives here:

    Compares produce a mask of all-ones in lanes where the predicate holds and all-zeros where it doesn't: vpcmpeqd / vpcmpgtd for integers (AVX2 has only equal and greater-than) and vcmpps with a predicate immediate for floats[9]. In SSE/AVX the mask occupies a regular vector register; in AVX-512 it lands in a separate K mask register (more on that below).
    vblendvps selects between two vectors lane by lane based on the sign bit of a third vector: a floating-point "if (predicate) take B else take A." Two μops on Haswell and Skylake, three on Alder Lake's P-cores, one on Zen 2 and later[1].

Worked example

conditionalLerp with alpha = 0.25, on two lanes that both hold 8 in destination and 4 in overrideValue. Both lanes compute the blend: 8 × 0.75 + 4 × 0.25 = 7.

Lane A's maskArray entry is 1, so the select keeps the 7. Lane B's is 0, so its compare lane is all ones and the select takes the original 8 instead, discarding the 7 it computed.

This does strictly more work than the scalar version: every lane computes the blended result even when its mask is zero, and the blend throws that result away. The win comes from processing eight elements per iteration and never paying a branch mispredict. The tradeoff resembles the branchy-vs-branchless story from the Assembly tutorial, with one difference: the eight-wide processing usually pays for the wasted lanes even when the predicate is predictable. The exception is a heavy body behind a predicate that is almost always false, where skipping the work outright (a scalar branch, or a "does any lane need this?" test on the whole vector) wins.
AVX-512: first-class lane masking

The story changes substantially on AVX-512. Every arithmetic instruction has a masked variant that operates only on lanes where a mask register is set; lanes where the mask is clear either keep their old value (merge-masking) or get zeroed (zero-masking)[7]. There are eight mask registers, K0 through K7. The EVEX encoding (AVX-512's 4-byte instruction prefix) has a 3-bit mask-selector field; the value 0 in that field is reserved to mean "no mask" (every lane active), which makes K0 unusable as an actual writemask. K0 still exists as a general mask register for KMOV, KAND, and the other mask-manipulation instructions; you just can't name it as the writemask on a masked vector op[35].
clamp_avx512.cpp · the same kernel with native masking

void clampZeroOneAvx512(float* inputArray, int elementCount) {
  __m512 zeroVector = _mm512_setzero_ps();
  __m512 oneVector  = _mm512_set1_ps(1.0f);

  int i = 0;
  for (; i + 16 <= elementCount; i += 16) {
    __m512 inputVector = _mm512_loadu_ps(inputArray + i);
    __m512 inRange = _mm512_min_ps(_mm512_max_ps(inputVector, zeroVector), oneVector);
    _mm512_storeu_ps(inputArray + i, inRange);
  }

  // Tail: no scalar loop needed. A mask of (1 << remaining) - 1 enables only the live lanes.
  if (i < elementCount) {
    __mmask16 tailMask = (__mmask16)((1u << (elementCount - i)) - 1u);
    __m512 inputVector = _mm512_maskz_loadu_ps(tailMask, inputArray + i);   // inactive lanes read as 0
    __m512 inRange     = _mm512_min_ps(_mm512_max_ps(inputVector, zeroVector), oneVector);
    _mm512_mask_storeu_ps(inputArray + i, tailMask, inRange);                // store only active lanes
  }
}

Worked example

Clamp 503 floats with clampZeroOneAvx512. The 16-wide loop runs 31 times and covers 496 elements, leaving 503 − 496 = 7.

The tail mask is (1 << 7) − 1 = 127, binary 0000 0000 0111 1111: lanes 0 to 6 on, 7 to 15 off. The masked load reads 7 floats, the masked store writes 7, and nothing past the end of the array is touched.

Three things AVX-512's mask model makes cheap that pre-AVX-512 code paid for explicitly:

    Tail handling. The masked load and store make the scalar tail loop go away. Element counts that aren't multiples of the SIMD width stop being a special case. (AVX already had vmaskmovps for masked loads and stores, but the mask lives in a vector register and the store form is microcoded on Zen 2 through Zen 4, about 40 μops each[1]; AVX-512 makes masking part of every instruction.)
    Sparse updates. "Update only the entities where isActive" becomes a single masked store; lanes whose isActive bit is zero don't write to memory, so there is no load-blend-store round trip.
    Conditional FMA. vfmadd231ps zmm0 {k1}, zmm1, zmm2 updates only the lanes where k1 is set; the other lanes keep their old value. This is a true predicated SIMD instruction, the way ARM SVE works natively.

The widget below runs vfmadd231ps zmm0 {k1}, zmm1, zmm2 across sixteen lanes. The mask register is the row of 0/1 bits at the top; lanes where the bit is 1 compute zmm0 + zmm1 × zmm2, lanes where it's 0 keep their old value. The pattern buttons change the mask for the same inputs; Step feeds new inputs and runs the FMA again, accumulating into zmm0:
Live · AVX-512 mask register predication
All lanes active
Every other lane
Lower half only
Random pattern
Step ▶
The top row is the mask k1; lanes with a 1 take zmm0 + zmm1 × zmm2, and lanes with a 0 keep their old zmm0 (blue). In the starting pattern lane 7 becomes 9 + 4 × 4 = 25, while lane 6 stays 2 even though its product, 1 × 2, isn't zero. Try the other patterns, then Step to run the FMA again on new inputs. Merge-masking is the default: lanes where the mask is 0 retain their old destination value. The zero-masking variant (intrinsics named _mm512_maskz_*) writes 0 in inactive lanes instead. On Skylake-X and later a masked arithmetic instruction is still one μop with the same latency as the unmasked form[1]. Merge-masking does make the old destination an input, so it joins the dependency chain; zero-masking doesn't.

[Back to SIMD from Scratch](index.md)
