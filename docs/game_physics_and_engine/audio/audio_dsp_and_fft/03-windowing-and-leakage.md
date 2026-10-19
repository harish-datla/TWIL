# Windowing and Leakage

Taking a finite block implicitly multiplies a longer signal by a rectangular window. Unless the block is exactly periodic, its endpoints jump, spreading energy into nearby bins: spectral leakage. The rectangular window’s first sidelobe is only about −13 dB.

Hann, Hamming, and Blackman windows taper the block toward zero before the FFT. There is no universally best window: lower sidelobes reduce leakage but widen the main lobe and reduce frequency resolution.

| Window | First sidelobe | Approximate main-lobe width |
| --- | ---: | ---: |
| Rectangular | −13 dB | 2 bins |
| Hann | −31 dB | 4 bins |
| Hamming | −43 dB | wider |
| Blackman | −58 dB | about 6 bins |

Use the periodic/DFT-even form for spectral analysis rather than the symmetric form used for FIR design. Windowing reduces leakage; it never eliminates it. A Hann window is a useful default when a balanced tradeoff is wanted.
