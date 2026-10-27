# Resampling

A 44.1 kHz asset played on a 48 kHz device, or any pitched playback, requires values at fractional source positions. The quality ladder is nearest-neighbor, linear interpolation, and windowed-sinc/polyphase filtering.

For 44.1 → 48 kHz, each output sample advances `44,100 / 48,000 = 0.91875` source samples. Output sample 10 is at source position 9.1875, so linear interpolation is:

```text
0.8125 × source[9] + 0.1875 × source[10]
```

Linear interpolation is a weak low-pass (`sinc²`) and allows significant spectral images to alias back into the audible band. Production resamplers use windowed-sinc or polyphase filters. Pitch-by-resampling also changes duration and formants; preserving them requires a separate pitch-shifting algorithm.
