# A Worked Engine Kernel: Batched 4x4 Matrix-Vector Transform


A particle system simulates in its emitter's local space, and every frame its particles have to be moved into world space by the same 4×4 matrix before they're drawn. Renderers, physics, and animation code all apply 4×4 transforms to large batches of points. The scalar form is sixteen multiplies and twelve adds per point. The SIMD form depends on where the parallelism comes from:

    One vertex at a time, 128 bits. Each vec4 is one __m128; the 4×4 matrix-vector product is four broadcasts, a multiply, and three FMAs (the form from the Assembly tutorial's §13, where a broadcast copies one value into every lane). One vertex per kernel call; useful when points arrive one at a time. Throughput-poor on AVX2 because half of each 256-bit operation goes unused.
    Eight vertices at a time, AoSoA, 256 bits. The data layout the throughput-tuned form needs: 8 vertices laid out as (x[0..7], y[0..7], z[0..7], w[0..7]). One YMM register per component. The transform broadcasts each matrix entry to all eight lanes and FMAs across all eight vertices in parallel.

The 8-at-once form suits any stage that pushes many points through one matrix, such as instance transforms or particle output. The kernel is short:
matvec_batch.cpp · transform 8 vertices by one matrix per AVX2 iteration

// AoSoA block: 8 vertices laid out (x[0..7], y[0..7], z[0..7], w[0..7]).
struct alignas(32) VertexBlock8 {
  float xCoordinates[8];
  float yCoordinates[8];
  float zCoordinates[8];
  float wCoordinates[8];   // usually 1.0f for positions, 0.0f for direction vectors
};

struct Matrix4x4 { float m[16]; };   // row-major: m[row * 4 + column]

void transformVertexBlock(const Matrix4x4& transform, VertexBlock8& block) {
  // Load each input component (8 vertices' x in one ymm register, etc.). One vmovaps each.
  __m256 xLanes = _mm256_load_ps(block.xCoordinates);
  __m256 yLanes = _mm256_load_ps(block.yCoordinates);
  __m256 zLanes = _mm256_load_ps(block.zCoordinates);
  __m256 wLanes = _mm256_load_ps(block.wCoordinates);

  // Output x = m00*x + m01*y + m02*z + m03*w, applied to all 8 vertices in parallel.
  // Each broadcast (vbroadcastss) replicates one matrix entry across all 8 lanes.
  __m256 outputX = _mm256_mul_ps  (_mm256_set1_ps(transform.m[ 0]), xLanes);
  outputX        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 1]), yLanes, outputX);
  outputX        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 2]), zLanes, outputX);
  outputX        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 3]), wLanes, outputX);

  __m256 outputY = _mm256_mul_ps  (_mm256_set1_ps(transform.m[ 4]), xLanes);
  outputY        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 5]), yLanes, outputY);
  outputY        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 6]), zLanes, outputY);
  outputY        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 7]), wLanes, outputY);

  __m256 outputZ = _mm256_mul_ps  (_mm256_set1_ps(transform.m[ 8]), xLanes);
  outputZ        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[ 9]), yLanes, outputZ);
  outputZ        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[10]), zLanes, outputZ);
  outputZ        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[11]), wLanes, outputZ);

  __m256 outputW = _mm256_mul_ps  (_mm256_set1_ps(transform.m[12]), xLanes);
  outputW        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[13]), yLanes, outputW);
  outputW        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[14]), zLanes, outputW);
  outputW        = _mm256_fmadd_ps(_mm256_set1_ps(transform.m[15]), wLanes, outputW);

  _mm256_store_ps(block.xCoordinates, outputX);
  _mm256_store_ps(block.yCoordinates, outputY);
  _mm256_store_ps(block.zCoordinates, outputZ);
  _mm256_store_ps(block.wCoordinates, outputW);
}

The work per vertex is exactly the same as the scalar form: four output components, each a four-term dot product against a matrix row. The difference is that every FMA does eight in parallel. The kernel has four independent output chains (one per output component), each four ops deep (one multiply + three FMAs), so its critical-path latency is roughly 4 ops × 4-cycle FMA latency = 16 cycles. In a loop over many blocks the out-of-order core overlaps consecutive blocks and the throughput limit takes over: 16 multiply/FMA ops per block at 2 per cycle is 8 cycles per block. With the broadcasts inside the loop as written, the 16 broadcast loads plus 4 vertex loads (20 loads at 2 per cycle) push that to about 10 cycles. The realistic range is 8 to 20 cycles per block of 8, or roughly 1 to 2.5 cycles per vertex, depending on whether the broadcasts are hoisted and how well the surrounding code lets blocks overlap.

Worked example

A matrix that moves points 10 units along x: the identity, with m[3] = 10. For the vertex (1, 2, 3, 1), output x = 1·1 + 0·2 + 0·3 + 10·1 = 11, and y, z and w come out as 2, 3 and 1. Lane k of outputX is that same row-times-vertex sum for vertex k, eight vertices at once.

Cost: a block needs 16 multiply/FMA ops, 8 cycles at two per cycle, so 1 cycle per vertex; with the broadcasts reloaded per block, 20 loads at two per cycle make it 10 cycles, 1.25 per vertex. A scalar version with FMA spends the same 16 ops on every single vertex: 8 cycles per vertex.

What this code intentionally skips. The matrix broadcasts are reloaded every call: a real engine hoists them out of the inner loop (broadcast the matrix into 16 registers once, transform many vertex blocks against it). The batching choice depends on whether you're skinning (many matrices, few vertices each) or instancing (one matrix, many vertices each); the form above is the instancing form. Skinning's per-vertex bone lookup needs the gather pattern from §10. DirectXMath's XMVector3TransformStream shows both approaches in one function: its SSE path runs the one-vertex broadcast form four vertices per iteration, while its NEON path deinterleaves four vertices into SoA registers with vld3q_f32 and runs the batch form[19].

[Back to SIMD from Scratch](index.md)
