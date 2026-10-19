# The Audio Engine

Rendering is soft real-time: a late frame stutters. Audio is hard real-time: a buffer that is not ready when the device needs it becomes a click or dropout. This tutorial builds an audio subsystem in C++ and Rust with a real-time-safe callback, lock-free command ring, mixer, limiter, resampler, panning, and distance attenuation.

Time: ~55 min  
Level: Senior  
Prerequisites: Lock-free Queues and The Game Loop are helpful.  
Stack: C++ (miniaudio) · Rust (cpal)

## What you’ll be able to do

- Write a callback that never allocates, locks, blocks, or performs I/O.
- Choose a buffer size by trading latency against underrun risk.
- Send play commands through a lock-free ring without freeing on the audio thread.
- Mix with headroom and a limiter instead of clipping.
- Resample, pan with equal power, and attenuate by distance.

## Sections

- [The Real-Time Callback](01-the-real-time-callback.md)
- [PCM Fundamentals](02-pcm-fundamentals.md)
- [Buffer and Latency](03-buffer-and-latency.md)
- [The Ring to the Audio Thread](04-the-ring-to-the-audio-thread.md)
- [Mixing](05-mixing.md)
- [Resampling](06-resampling.md)
- [Spatial Audio](07-spatial-audio.md)
- [APIs and Libraries](08-apis-and-libraries.md)
- [Pitfalls](09-pitfalls.md)
- [What’s Next](10-whats-next.md)
