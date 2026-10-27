# The DFT and the FFT

The DFT of `N` samples produces `N` complex bins. For bin `k`:

```text
X[k] = Σ(n=0..N−1) x[n] × e^(−i 2πkn/N)
```

Magnitude is `sqrt(re² + im²)` and phase is `atan2(im, re)`. A direct DFT costs `O(N²)`. The Cooley–Tukey FFT computes exactly the same result in `O(N log N)` by splitting even and odd samples and combining them with twiddle-factor butterflies.

A 1024-sample block at 48 kHz has bin spacing `48,000/1,024 = 46.875 Hz`, with 513 independent real-input bins from 0 to 24 kHz. A 440 Hz tone falls between bins 9 and 10. A direct transform takes over one million complex multiply-adds; a radix-2 FFT takes about 5,120 butterflies.

```cpp
void fftRadix2(std::vector<std::complex<float>>& samples) {
    size_t n = samples.size();
    for (size_t i = 1, j = 0; i < n; ++i) {
        size_t bit = n >> 1;
        for (; j & bit; bit >>= 1) j ^= bit;
        j ^= bit;
        if (i < j) std::swap(samples[i], samples[j]);
    }
    for (size_t len = 2; len <= n; len <<= 1) {
        float angle = -2.0f * 3.14159265f / float(len);
        std::complex<float> step(std::cos(angle), std::sin(angle));
        for (size_t start = 0; start < n; start += len) {
            std::complex<float> w(1, 0);
            for (size_t k = 0; k < len / 2; ++k) {
                auto even = samples[start+k];
                auto odd = samples[start+k+len/2] * w;
                samples[start+k] = even + odd;
                samples[start+k+len/2] = even - odd;
                w *= step;
            }
        }
    }
}
```

This teaching implementation requires a power-of-two size and uses a full complex transform. Production code should use FFTW, KissFFT, rustfft, or a real-FFT routine. Mixed-radix and Bluestein algorithms cover other sizes. Do not claim all `N` bins are independent for real input, and retain phase when reconstructing.
