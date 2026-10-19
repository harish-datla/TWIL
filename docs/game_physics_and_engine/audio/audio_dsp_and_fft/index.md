# Audio DSP & the FFT

A gunshot behind a wall should sound muffled, a music visualizer needs to know how much bass is playing, and a dry footstep should sound like it was recorded in a stone corridor. These are signal-processing operations on the samples produced by the mixer. This tutorial covers FFT analysis, windowing, spectrograms, filters, convolution reverb, and real-time DSP scheduling.

Time: ~55 min  
Level: Senior  
Prerequisites: [The Audio Engine](../audio_engine/index.md) and SIMD.  
Stack: C++ & Rust

## What you’ll be able to do

- Turn a block of samples into a spectrum and identify each bin’s frequency.
- Choose a window and block length while understanding leakage and time/frequency tradeoffs.
- Implement an RBJ biquad, choose its type, cutoff, and Q, and keep it stable.
- Explain FFT convolution, partitioning, and when algorithmic reverb is preferable.
- Lay out per-voice and per-bus effects inside a real-time deadline.

## Sections

- [Two Domains](01-two-domains.md)
- [The DFT and the FFT](02-the-dft-and-the-fft.md)
- [Windowing and Leakage](03-windowing-and-leakage.md)
- [The STFT and Spectrogram](04-the-stft-and-spectrogram.md)
- [Digital Filters](05-digital-filters.md)
- [Convolution Reverb](06-convolution-reverb.md)
- [Real-Time Game DSP](07-real-time-game-dsp.md)
- [Pitfalls](08-pitfalls.md)
- [What’s Next](09-whats-next.md)
