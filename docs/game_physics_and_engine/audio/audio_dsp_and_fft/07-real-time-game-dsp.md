# Real-Time Game DSP

DSP runs in the hard-real-time callback or in worker threads feeding it: no heap allocation, locks, I/O, or unbounded work. Pre-allocate filter and reverb state, pass parameter changes through the lock-free command ring, and process whole blocks for cache and SIMD efficiency.

Cheap per-voice effects such as distance or occlusion low-pass filters run on each voice. Expensive shared effects—reverb, master EQ, and limiting—run once on a bus. SIMD helps independent loops such as FFT butterflies, magnitude calculations, and parallel voices. A straightforward biquad recurrence is serial in time, so vectorize across voices or channels instead.

Smooth parameter changes by ramping cutoff or interpolating coefficients across a block. Jumping parameters once per block can create zipper noise. Sixty-four one-pole filters at 48 kHz cost about 3.07 million multiply-adds per second, while putting a convolution reverb on every voice would multiply its work unnecessarily; send the voices to one reverb bus.
