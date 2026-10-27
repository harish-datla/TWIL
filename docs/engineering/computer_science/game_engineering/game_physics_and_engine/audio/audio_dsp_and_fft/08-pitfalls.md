# Pitfalls

- **“The FFT is approximate”:** It computes the DFT exactly; it is an efficient algorithm.
- **Treating all `N` real-input bins as independent:** Only `N/2+1` are independent; the rest mirror them.
- **“Hann is always best”:** Window choice trades sidelobes against main-lobe width.
- **Assuming a longer FFT is strictly better:** It improves frequency resolution while worsening time resolution.
- **Calling FIR or IIR universally superior:** FIR costs taps; IIR is cheap but can ring or become unstable.
- **Using one high-order IIR section:** Cascade biquads for stability and coefficient robustness.
- **Assuming FFT convolution is always faster:** Short FIRs are cheaper directly, and unpartitioned FFT convolution adds latency.
- **Allocating or locking in DSP:** Pre-allocate, use the command ring, and handle denormals.
