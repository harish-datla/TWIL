# Sources & Further Reading


Numbered citations refer to the superscripts above. Most are freely available; a few are vendor manuals that require a no-cost registration.
A note on originality

The prose, code samples, CSS, and interactive widgets on this page are original writing. The lane-format and instruction-encoding details follow the SSE, AVX, and AVX-512 chapters of Intel SDM Volume 1 [2] and the Intel Intrinsics Guide [9]; latency, throughput, and μop counts come from uops.info [1]. The multi-accumulator rule follows from the latency and throughput figures in the AMD and Intel optimization guides [6][32]. The pshufb-as-lookup-table examples follow Wojciech Muła's blog and the Keiser–Lemire UTF-8 validation paper [4][16]. The AVX-512 frequency history in §14 follows Travis Downs's measurements and Phoronix's server benchmarks [26][36]; the SoA/AoSoA framing in §4 echoes Mike Acton's CppCon 2014 talk on data-oriented design [13]. The frustum cull kernel in §12 uses the standard sphere-plane te

[Back to SIMD from Scratch](index.md)
