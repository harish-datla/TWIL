# The Subnormal Cliff


The music stops, the reverb tail fades out, and the audio thread, which was fine during the loud part, suddenly misses its deadline and the sound crackles. On some CPUs a quiet signal costs more than a loud one, because as it decays it fills with subnormals.

On some hardware, arithmetic that produces or consumes subnormals drops into a microcode-assisted slow path that is dramatically slower than the same operation on normal numbers. Bruce Dawson measured a worst case of roughly a 175× slowdown for SSE addition and multiplication with subnormals on a Core 2, and about a 140-cycle penalty on a Sandy Bridge mulps, where he could produce no slowdown at all on addps[12]. The size depends on the CPU and the operation, so quote it with its CPU rather than as a universal number.

Worked example

A one-pole filter keeps a state and, once its input goes silent, multiplies that state by 0.99 every sample. From 1.0 the state drops below the smallest normal float, 1.18 × 10−38, after 8,690 samples, 0.18 seconds at 48 kHz. From then on every multiply touches a subnormal.

It never reaches zero. At 50 × 2−149 ≈ 7.0 × 10−44, multiplying by 0.99 gives just over 49.5 steps of 2−149, which rounds back up to 50, so the state sits there forever and every sample after the first 0.18 s of silence is a subnormal multiply. With flush-to-zero on, the 8,690th sample becomes exactly 0 instead.

The fix is FTZ (flush-to-zero, for outputs) and DAZ (denormals-are-zero, for inputs), two bits in the x86 MXCSR control register (the per-thread register that holds SSE's rounding mode and exception settings) that make the CPU treat subnormals as zero, trading correctness near zero for speed. It is a per-thread CPU mode, so set it on every audio or physics worker thread, not just the main one[13]. Rust has no sound equivalent: its compiler assumes the default MXCSR, flipping these bits is undefined behavior, and the _mm_setcsr intrinsic has been deprecated since Rust 1.75 for that reason[15]. Rust code keeps subnormals out of its feedback paths instead.
Enable FTZ and DAZ (x86)
C++
Rust

#include <xmmintrin.h>   // FTZ (flush-to-zero, outputs)
#include <pmmintrin.h>   // DAZ (denormals-are-zero, inputs; SSE3)
_MM_SET_FLUSH_ZERO_MODE(_MM_FLUSH_ZERO_ON);        // MXCSR bit 15
_MM_SET_DENORMALS_ZERO_MODE(_MM_DENORMALS_ZERO_ON); // MXCSR bit 6; call per thread

Where this actually bites: audio

Recursive filters and reverb tails decay exponentially toward zero, so when a signal goes quiet (a DAW, the music software hosting the plugin, stops playback and streams zeros into a reverb) they generate a flood of subnormals, and on an affected CPU the audio thread blows its budget and drops out. Enabling FTZ/DAZ on the audio thread is the standard mitigation in C++; flushing tiny values out of the filter state works everywhere. And recall the link-time footgun from §9: -ffast-math turns FTZ on process-wide, which is how most people enable this without meaning to[10].

[Back to IEEE-754 Floating Point](index.md)
