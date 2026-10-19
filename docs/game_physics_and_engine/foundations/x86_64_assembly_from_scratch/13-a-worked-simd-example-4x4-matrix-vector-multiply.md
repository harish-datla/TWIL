# A Worked SIMD Example: 4x4 Matrix-Vector Multiply


Culling, skinning and physics all move points between coordinate spaces, and each move is a matrix times a vector. The workhorse transform of 3D code: multiply a 4-component vector by a 4×4 matrix. Naively, sixteen multiplies and twelve adds (four dot products of length 4) per vertex. With SSE the same operation is four broadcasts plus four multiplies and three adds on 128-bit vectors, or one multiply and three FMAs. The standard formulation, for a column-major matrix M = [c0 | c1 | c2 | c3] and a vector v = (x, y, z, w):
Mv = c0·x + c1·y + c2·z + c3·w

In words: the result is each column of the matrix scaled by the matching component of the vector, then all four added up. That's four "vector times one number" steps and three vector adds, which is exactly the shape SIMD registers are good at.

Worked example

A translation by (10, 20, 30): columns c0 = (1, 0, 0, 0), c1 = (0, 1, 0, 0), c2 = (0, 0, 1, 0), c3 = (10, 20, 30, 1). Transform the point v = (1, 2, 3, 1).

c0·1 + c1·2 + c2·3 + c3·1 = (1, 0, 0, 0) + (0, 2, 0, 0) + (0, 0, 3, 0) + (10, 20, 30, 1) = (11, 22, 33, 1).

Each ci·s is a scalar-times-vector, which in SSE is a broadcast of the scalar across four lanes followed by a multiply. The three additions chain together. Source:
matvec.cpp · 4x4 column-major mat * vec, SSE + FMA3

// 16-byte aligned so the columns can be loaded with movaps. col[i] is column i of M.
struct alignas(16) mat4 { __m128 col[4]; };

__m128 mul(const mat4& matrix, __m128 vec) {
  // Broadcast each component of vec into all four lanes.
  __m128 splatX = _mm_shuffle_ps(vec, vec, _MM_SHUFFLE(0, 0, 0, 0));
  __m128 splatY = _mm_shuffle_ps(vec, vec, _MM_SHUFFLE(1, 1, 1, 1));
  __m128 splatZ = _mm_shuffle_ps(vec, vec, _MM_SHUFFLE(2, 2, 2, 2));
  __m128 splatW = _mm_shuffle_ps(vec, vec, _MM_SHUFFLE(3, 3, 3, 3));
  __m128 result = _mm_mul_ps(matrix.col[0], splatX);          // c0 * x
  result = _mm_fmadd_ps(matrix.col[1], splatY, result);          // + c1 * y   (FMA3)
  result = _mm_fmadd_ps(matrix.col[2], splatZ, result);          // + c2 * z
  result = _mm_fmadd_ps(matrix.col[3], splatW, result);          // + c3 * w
  return result;
}

With clang -O3 -mavx2 -mfma, the body is eight or nine instructions plus the return, depending on the version. The listing below is eight; Clang 18, for example, broadcasts x with vbroadcastss, uses the 231 form for the last FMA and adds a final vmovaps to move the result into xmm0:
matvec.s · illustrative clang -O3 -mavx2 -mfma output

mul(mat4 const&, __m128):
  ; rdi = const mat4*  (the matrix);  xmm0 = v
  vshufps      xmm1, xmm0, xmm0, 0x00       ; xmm1 = {v.x, v.x, v.x, v.x}
  vshufps      xmm2, xmm0, xmm0, 0x55       ; xmm2 = {v.y, v.y, v.y, v.y}
  vshufps      xmm3, xmm0, xmm0, 0xAA       ; xmm3 = {v.z, v.z, v.z, v.z}
  vshufps      xmm0, xmm0, xmm0, 0xFF       ; xmm0 = {v.w, v.w, v.w, v.w} (last use of v)
  vmulps       xmm1, xmm1, [rdi + 0x00]     ; xmm1  = c0 * v.x
  vfmadd231ps  xmm1, xmm2, [rdi + 0x10]     ; xmm1 += c1 * v.y   (231 form: dst = src2*src3 + dst)
  vfmadd231ps  xmm1, xmm3, [rdi + 0x20]     ; xmm1 += c2 * v.z
  vfmadd132ps  xmm0, xmm1, [rdi + 0x30]     ; xmm0  = v.w * c3 + xmm1   (132 form lands the result in xmm0)
  ret

Two observations worth pulling out. The four multiply-adds form one serial chain: a multiply and three dependent FMAs, about 16 cycles of latency at Skylake's 4 cycles each. Writing the source as two chains, (c0·x + c1·y) + (c2·z + c3·w), cuts the critical path to three dependent operations (about 12 cycles) for one extra instruction. Compilers won't regroup it on their own without -ffast-math (specifically -fassociative-math), because the regrouped sum rounds differently. Second, the matrix columns are read directly by the multiply and FMA memory operands ([rdi + 0x10], etc.); the load and the arithmetic travel through the front end as one fused μop on Intel and one macro-op on AMD[13], so a separate vmovaps would only add work.

What's intentionally missing from this code. The 128-bit form processes one vertex at a time. The throughput-tuned form transposes the data to SoA so that one ymm/zmm register holds the same component (x, then y, then z) for many vertices, and a single broadcast-and-FMA transforms 8 vertices at a time with AVX or 16 with AVX-512. The matrix is also typically loaded once and reused across many vertices; this code reloads it every call. Per-vertex rendering transforms run on the GPU in vertex shaders, so CPU-side vector transforms mostly serve culling, physics, animation pose evaluation, and gameplay queries.

[Back to x86-64 Assembly from Scratch](index.md)
