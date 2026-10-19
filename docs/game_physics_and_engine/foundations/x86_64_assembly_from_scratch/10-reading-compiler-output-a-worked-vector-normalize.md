# Reading Compiler Output: A Worked Vector Normalize


Lighting, physics and camera code normalize vectors constantly, so it's worth knowing exactly what a normalize compiles to. A canonical 3-D engine primitive: take a vec3, divide each component by its length, return the unit vector. For (3, 0, 4): the squared length is 9 + 0 + 16 = 25, the length is 5, and the result is (0.6, 0, 0.8). Source:
normalize.cpp

struct vec3 { float x, y, z; };

vec3 normalize(vec3 v) {
  float lenSquared = v.x * v.x + v.y * v.y + v.z * v.z;
  float invLen    = 1.0f / sqrtf(lenSquared);
  return { v.x * invLen, v.y * invLen, v.z * invLen };
}

With clang -O2 -mavx2 -mfma -fno-math-errno on System V (illustrative; exact instruction selection varies across versions and surrounding code):
normalize.s · illustrative clang -O2 -mavx2 -mfma -fno-math-errno output · Intel syntax

normalize(vec3):
  ; System V vec3 ABI: xmm0[0]=x, xmm0[1]=y (upper lanes undefined); xmm1[0]=z.
  vmulss       xmm3, xmm0, xmm0          ; xmm3[0] = x*x
  vmovshdup    xmm2, xmm0                ; xmm2 = {xmm0[1], xmm0[1], …} → lane 0 holds y
  vfmadd231ss  xmm3, xmm2, xmm2          ; xmm3[0] += y*y   (one rounding for the multiply-add)
  vfmadd231ss  xmm3, xmm1, xmm1          ; xmm3[0] += z*z   → lenSquared
  vsqrtss      xmm3, xmm3, xmm3          ; xmm3[0] = sqrt(lenSquared)
  vmovss       xmm4, dword ptr [rip + .LC0] ; xmm4[0] = 1.0f from a read-only constant
  vdivss       xmm3, xmm4, xmm3          ; xmm3[0] = invLen
  vmulss       xmm0, xmm0, xmm3          ; xmm0[0] = x*invLen; xmm0[1..3] copied from the first source
  vmulss       xmm2, xmm2, xmm3          ; xmm2[0] = y*invLen
  vmulss       xmm1, xmm1, xmm3          ; xmm1[0] = z*invLen
  vunpcklps    xmm0, xmm0, xmm2          ; xmm0 = {x*invLen, y*invLen, …}
  ret                                    ; return {x', y'} in xmm0, z' in xmm1[0]

A few things are happening here that aren't obvious from the source. The two-XMM vec3 argument convention is from the System V ABI[6]: an aggregate of three floats has its x,y in xmm0 and z in xmm1, and the same two registers carry the result back. The compiler contracted x*x + y*y + z*z into one multiply and two vfmadd231ss (fused multiply-add, scalar single-precision), keeping the source's left-to-right order; an FMA does a += b*c in one μop with one rounding step instead of two[16], so this build's result can differ in the last bit from a build without FMA. The -fno-math-errno flag matters: without it, GCC and Clang on Linux keep a compare and a fallback call to sqrtf so that a negative input still sets errno. The vsqrtss / vdivss pair is the correctly rounded square root and reciprocal; hot paths sometimes replace it with vrsqrtss (an approximate reciprocal square root with about 12 bits of precision) plus one Newton-Raphson step, trading a little accuracy for lower latency. Compilers make that substitution only when told they may (fast-math flags, plus -mrecip on GCC), because it changes results.

Reading conventions: the v prefix on each mnemonic is the AVX (VEX-encoded) three-operand form. Without AVX the same multiply would be mulss xmm3, xmm0, where the destination is always also a source, so keeping x alive would cost an extra movaps to copy it first. Three-operand AVX writes the result to a separate register instead. The ss suffix means "scalar single-precision": one float in the low 32 bits of the XMM register, with the other three lanes passed through[17].

[Back to x86-64 Assembly from Scratch](index.md)
