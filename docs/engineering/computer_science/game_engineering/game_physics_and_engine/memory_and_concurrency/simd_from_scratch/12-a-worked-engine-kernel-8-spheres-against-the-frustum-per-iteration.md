# A Worked Engine Kernel: 8 Spheres against the Frustum per Iteration


A level holds thousands of objects, and the camera sees only a pyramid-shaped slice of it; anything entirely outside that slice shouldn't cost a draw call. Most renderers cull bounding volumes against the camera frustum (the six-sided volume the camera can see) before submitting draw calls, either on the CPU or, in GPU-driven renderers, in a compute pass. The scalar form is one sphere-vs-six-planes test per object; the SIMD form does eight at once and lets the predicate pass through the lane-mask machinery from §8.

The test asks one question per wall of the frustum: is the sphere entirely on the outside of it? Measure the signed distance from the sphere's center to the wall's plane, positive on the outside. If that distance is greater than the radius, the whole sphere is outside and the object is culled.

Worked example

A wall at x = 10 whose outside is +x: normal (1, 0, 0), offset d = −10, so the signed distance of a point is x − 10. Three spheres of radius 1:

Center (12, 0, 0): distance 2 > 1, fully outside, culled. Center (10.5, 0, 0): distance 0.5, between −1 and 1, so it straddles the wall and is kept. Center (5, 0, 0): distance −5 < −1, fully inside this wall.

In general, a frustum plane is the equation nx · x + ny · y + nz · z + d = 0 with n the unit outward normal and d the plane's offset from the origin. A point lies outside the plane when nx · px + ny · py + nz · pz + d > 0. For a bounding sphere centered at (px, py, pz) with radius r, the sphere is fully outside the plane when that same dot product is greater than r; it's fully inside when the dot product is less than -r; it straddles otherwise. The standard cull keeps any object that isn't fully outside any of the six planes[20].

The SoA form has all eight spheres' xs in one register, all eight ys in the next, eight zs, eight radii. One plane test is three FMAs (the plane offset rides in as the first addend), a compare, and an and-not; six planes is eighteen FMAs, six compares, and six and-nots, the same instruction count the scalar form spends on a single sphere.
frustum_cull.cpp · 8 spheres × 6 planes per AVX2 iteration

struct alignas(32) SphereBatch8 {
  float centerX[8], centerY[8], centerZ[8], radius[8];
};

struct FrustumPlane { float normalX, normalY, normalZ, distance; };

// Returns an 8-bit mask: bit i is 1 if sphere i passed (was not culled), 0 if outside.
int cullBatchAgainstFrustum(const SphereBatch8& spheres, const FrustumPlane planes[6]) {
  __m256 centerXVector = _mm256_load_ps(spheres.centerX);
  __m256 centerYVector = _mm256_load_ps(spheres.centerY);
  __m256 centerZVector = _mm256_load_ps(spheres.centerZ);
  __m256 radiusVector  = _mm256_load_ps(spheres.radius);

  // Start with all 8 visible; lanes that fail any plane get masked off.
  __m256 visibleMask = _mm256_castsi256_ps(_mm256_set1_epi32(-1));

  for (int p = 0; p < 6; ++p) {
    // Broadcast one plane's coefficients to all 8 lanes.
    __m256 planeNormalX = _mm256_set1_ps(planes[p].normalX);
    __m256 planeNormalY = _mm256_set1_ps(planes[p].normalY);
    __m256 planeNormalZ = _mm256_set1_ps(planes[p].normalZ);
    __m256 planeOffset  = _mm256_set1_ps(planes[p].distance);

    // signedDistance[i] = n·center[i] + d, computed in parallel for 8 spheres.
    __m256 signedDistance = _mm256_fmadd_ps(planeNormalX, centerXVector, planeOffset);
    signedDistance        = _mm256_fmadd_ps(planeNormalY, centerYVector, signedDistance);
    signedDistance        = _mm256_fmadd_ps(planeNormalZ, centerZVector, signedDistance);

    // A sphere is fully outside this plane if signedDistance > radius. Mask off those lanes.
    __m256 outsidePlane = _mm256_cmp_ps(signedDistance, radiusVector, _CMP_GT_OQ);
    visibleMask = _mm256_andnot_ps(outsidePlane, visibleMask);    // clear lanes where outsidePlane
  }
  // Extract the 8 sign bits as a single int; bit i is 1 if sphere i is still visible.
  return _mm256_movemask_ps(visibleMask);
}

Six planes × (three FMAs + one compare + one and-not) is thirty μops per batch of eight spheres, plus the broadcasts. The 24 FMAs and compares share Skylake's two FMA-capable ports (the and-nots can go to a third), so a batch takes on the order of 12 to 15 cycles, about 2 cycles per sphere[1]. A branch-free scalar version runs about the same thirty μops for each sphere, so the vector form's advantage approaches the lane count, 8×. Scalar code that exits after the first failing plane narrows the gap on scenes where most objects are culled early, and gives some of it back in branch mispredicts. The widget runs both:
Live · Frustum cull race (scalar vs AVX2)
Scalar (1 sphere / tick)
AVX2 (8 spheres / tick)
Run ▶
Reset ⟲
spheres processed
0
cycles (modelled)
0
cycles per sphere
···
Each circle is a bounding sphere seen from above, and the wedge is the frustum, apex on the right. Run each mode: circles turn green or red as they're tested, and over the same 240 spheres the scalar mode reports 3,600 modelled cycles (15.0 per sphere) against 450 (1.9 per sphere) for AVX2. Simplified Skylake model: thirty μops per test (three FMAs, a compare, and an and-not per plane, six planes) issued two per cycle, so 15 cycles per batch of eight in AVX2 mode and 15 cycles per sphere in scalar mode. The 2D scene shows only the two side planes of the wedge; a sphere stays green if any part of it is inside both. Real kernels add things this demo skips: an early exit once a plane has culled all eight lanes (a vmovmskps and a branch), and a hierarchical cull above the per-object pass[20].

[Back to SIMD from Scratch](index.md)
