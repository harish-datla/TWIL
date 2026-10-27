# Two Domains

A waveform shows amplitude over time but does not make it easy to distinguish bass rumble from hiss. The same sampled signal has a time-domain representation and a frequency-domain representation: sinusoidal components with amplitude and phase that sum to the waveform. The DFT is the exact, invertible bridge between them.

A 440 Hz tone of amplitude 0.5 plus a 1320 Hz tone of amplitude 0.2 looks complicated in 1024 time samples at 48 kHz, but appears as two spectral peaks. A visualizer can read bass energy directly from the low-frequency peak. The transform is a change of coordinates, not a lossy summary; perceptual codecs use related filterbanks for different reasons.
