# Convolution Reverb

Convolution reverb applies a measured room impulse response to a dry sound. Direct convolution of an input with an `M`-sample impulse response costs `O(NM)`. The convolution theorem turns it into FFTs, frequency-domain multiplication, and an inverse FFT; zero-pad to at least `N+M−1` so circular convolution equals linear convolution.

A 2-second IR at 48 kHz has 96,000 samples. Direct convolution would require about 4.6 billion multiply-adds per second for one channel. FFT convolution is vastly cheaper for a long IR, but processing blocks as long as the IR adds seconds of latency.

Partitioned convolution splits the impulse response into blocks, uses overlap-add, and sums delayed partitions. Non-uniform schemes use small early partitions for low latency and larger late partitions for efficiency. A short impulse response is often faster as a direct FIR. Algorithmic reverbs such as Schroeder networks use comb delays and all-pass filters; they add little latency, are cheap and parameterizable, but can sound less like a measured room.
