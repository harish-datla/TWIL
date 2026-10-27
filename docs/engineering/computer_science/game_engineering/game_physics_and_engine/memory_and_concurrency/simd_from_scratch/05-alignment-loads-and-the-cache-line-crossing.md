# Alignment, Loads, and the Cache-Line Crossing


Your SoA arrays come from a plain new float[count], and the same loop runs a little faster or slower depending on where the allocator happened to put them. The difference is where the loads fall relative to cache lines. Three load instructions cover almost all SIMD memory access[9]:

    _mm256_loadu_ps (vmovups in the disassembly): load eight floats from any address. No alignment requirement.
    _mm256_load_ps (vmovaps): load eight floats from a 32-byte-aligned address. Faults if the address isn't aligned, which makes it a cheap runtime check that the layout invariant holds; when the address is aligned, the two forms perform identically on AVX-era hardware[10].
    _mm256_stream_ps (vmovntps): a non-temporal store that bypasses the cache and requires a 32-byte-aligned address. Used when writing a large block that you know won't be read again before it's evicted (frame buffer composition, large texture uploads, particle render-output buffers). Wrong by default; right occasionally.

The performance cost that does show up in practice is the cache-line crossing. An x86 L1 cache line is 64 bytes. A 32-byte load that starts at offset 48 within a line straddles the boundary and needs a second L1 access. On Skylake-class cores a split-line load uses a load port twice, halving the load throughput for that μop and adding a few cycles of latency[10]. In a hot loop that issues two loads per cycle, a stream of split-line loads can cut inner-loop throughput nearly in half. The cost comes from the line crossing itself, and vmovups pays it wherever the address happens to fall.

Worked example

Plain malloc on x86-64 guarantees 16-byte alignment. Say it returns an address 16 bytes past a 64-byte line boundary. A loop of 32-byte loads then starts them at line offsets 16, 48, 16, 48, … The load at 48 covers bytes 48 to 79 and crosses into the next line, so half the loads split.

Align the array to 32 bytes and the loads start at offsets 0 and 32 only, covering bytes 0 to 31 and 32 to 63: none cross.

Tools to control alignment in C++[14]:
alignment.cpp · the toolkit

// 1) Type-level alignment: every variable of this type gets the wider alignment.
struct alignas(32) ParticleBlock {
  float positionX[8];
  float positionY[8];
  float positionZ[8];
};

// 2) Variable-level: a single declaration gets the wider alignment.
alignas(64) float buffer[1024];

// 3) Heap allocation: plain malloc and new guarantee 16 bytes on x86-64 with glibc
//    and the Windows CRT. For wider alignment, ask for it explicitly.
//    std::aligned_alloc (C++17; the size must be a multiple of the alignment):
float* page = static_cast<float*>(std::aligned_alloc(32, 1024 * sizeof(float)));
std::free(page);
//    MSVC doesn't provide std::aligned_alloc; use _aligned_malloc / _aligned_free there.

//    new of an over-aligned type picks the aligned operator new automatically (C++17):
ParticleBlock* blocks = new ParticleBlock[128];   // 32-byte aligned; delete[] matches
delete[] blocks;

//    For a plain float buffer, call the aligned allocation function directly
//    and free it with the matching aligned delete:
float* page2 = static_cast<float*>(::operator new[](1024 * sizeof(float), std::align_val_t{32}));
::operator delete[](page2, std::align_val_t{32});

// 4) Telling the compiler an existing pointer is aligned. C++20 has a portable form;
//    older compilers use __builtin_assume_aligned (GCC/Clang) or __assume (MSVC).
float* alignedPointer = std::assume_aligned<32>(rawPointer);
// The compiler may now use aligned loads and skip a peel loop that reaches alignment.

Three practical alignment rules:

    Align your big arrays to the SIMD width you target. 32 bytes for AVX/AVX2, 64 for AVX-512. Use alignas on the type or an aligned allocation. Don't hand-pad with extra bytes; the padding stops being right the moment someone changes the struct.
    Use loadu by default. On aligned data it costs nothing extra (both instructions decode to the same μop). On misaligned data the cache-line-crossing penalty is paid only by the loads that actually cross a line, not by every loadu. Reach for load when alignment is an invariant you want checked: a misaligned pointer faults at the load instead of quietly running slower.
    Pad your arrays to a multiple of the SIMD width. A loop over a 503-element array running 8 floats at a time has to handle the last 7 elements specially: either a scalar tail, an AVX-512 masked load, or a deliberate over-read into a padded buffer. Padding the array to 504 (a multiple of 8) and zeroing the tail is the cheapest of the three, at the cost of a few bytes of extra memory. (503 = 62 × 8 + 7: 62 full iterations cover 496 elements, and padding adds one float, 4 bytes.)


[Back to SIMD from Scratch](index.md)
