# Measure what actually ran

A smaller model file doesn't guarantee a faster run. A runtime can fall back to a higher-precision kernel for any operator it has no low-bit kernel for. Check which backend and kernels were selected before you benchmark. If a range or an axis is wrong, compare intermediate values layer by layer before you start tuning the sampler.

When you report a benchmark, list the hardware, runtime version, batch size, prompt and output lengths, warmup, and whether data transfers and tokenization are included. Keep memory occupied (bytes sitting allocated) separate from memory traffic (bytes read per token): a weight matrix occupies its memory once but is read again for every token.
A toy error metric is not a deployment acceptance test

The widgets compute exact errors for the arrays they show. A model needs its own accuracy and robustness evaluation, including inputs that push past the calibration ranges. How much error is acceptable depends on the task.
