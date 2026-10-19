# What's Intentionally Missing, and What's Next


This page stayed on bit work in single integers. Several neighbors were left out on purpose. SIMD bit operations (operating on 128, 256, or 512 bits at once, including the pshufb byte-shuffle as a parallel lookup) live in the SIMD tutorial. Bit-reversal permutations and the butterfly addressing behind the FFT are an audio and signal-processing topic. Gray codes, where consecutive values differ in one bit, matter for rotary encoders and some error-tolerant schemes. Entropy coders like rANS pack fractional bits for compression. Constant-time bit tricks for cryptography deliberately avoid data-dependent branches and timing, a discipline of its own.

Next in the series, C++ Containers moves from the bits in one word to how whole arrays, lists and maps sit in memory, and why that layout decides their speed. Later pages put these bits to work: the hash tables tutorial turns hashes into slots with Fibonacci hashing (a multiply and a right shift) and the power-of-two mask h & (m - 1); the spatial partitioning tutorial builds the Morton-ordered octrees from §12; and the memory allocators tutorial uses the alignment mask ((p + a - 1) & ~(a - 1)) that rounds a pointer up to a power-of-two boundary. 

[Back to Bit Shifting from Scratch](index.md)
