# Buffer and Latency

The callback buffer sets both latency and deadline. At 48 kHz, 256 frames are 5.33 ms; 64 frames are 1.33 ms. Smaller buffers respond sooner but leave less time to mix, so a USB stall, scheduler interruption, or over-budget callback can cause an underrun.

A sound triggered just after a callback begins waits up to one full buffer: up to 5.33 ms at 256 frames or 21.3 ms at 1024. The operating-system mixer and device can add more buffering. Halving a buffer halves that wait but also halves the execution window: a 3 ms stall is safe at 256 frames and unsafe at 128.

Choose a size that meets latency goals with margin under realistic scheduling jitter. Treat underrun robustness as a hard requirement, not an average benchmark. A larger buffer is often better than a tiny buffer that clicks under load.
