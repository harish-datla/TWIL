# Pitfalls

- **`malloc` or a lock in the callback:** Unbounded worst-case time misses the deadline. Use a lock-free ring.
- **Freeing on the audio thread:** Dropping the last `Arc`/`shared_ptr` can free memory there. Retire buffers to the game thread.
- **Dividing the mix by voice count:** It attenuates everything. Use headroom and a limiter.
- **Confusing audio and video frames:** An audio frame is one sample per channel.
- **Linear resampling of raw audio:** It aliases. Use an appropriate low-pass resampler.
- **Linear panning:** It creates a −3 dB center hole. Use equal-power with the fold-down caveat.
- **Pure `1/r²` distance:** It blows up at zero. Clamp a reference-distance rolloff.
- **Tiny buffers for low latency:** A tighter deadline increases underruns. Balance latency against margin.
