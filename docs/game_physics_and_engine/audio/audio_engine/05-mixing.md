# Mixing

Mixing is summation: add each gain-scaled voice into the output for every frame and channel. Independent voices often sum beyond `[-1,1]`. Clamping then hard-clips peaks and adds harmonic distortion.

Use headroom and a master limiter. A limiter is a high-ratio compressor with fast attack that reduces gain as the master approaches the ceiling while preserving more waveform shape. Do not divide by the number of voices: that makes a lone footstep quieter whenever other voices exist. Do not rely on hard clipping.

Three voices peaking at 0.5 can sum to 1.5, or `20 log₁₀(1.5) = 3.5 dB` above full scale. Dividing by four would make a footstep peaking at 0.8 become 0.2, 12 dB quieter just because other slots exist. Mix with roughly 6 dB headroom and let the limiter catch unusual pile-ups. Aggressive limiting can pump and flatten transients, so it is not free.
