# Reading the prompt is math, writing the answer is memory

Every request to a chatbot has two phases. First the model reads your whole prompt in one go, a phase called prefill. Then it writes the answer one token at a time, a phase called decode, and each new token needs another full pass through the model. The two phases hit different limits of the GPU, and that difference shapes everything a serving engine does.

Per token, both phases do about the same arithmetic: roughly 2 floating-point operations (FLOPs) per model parameter, one multiply and one add, plus some extra for attention. What differs is how many tokens share one read of the weights from memory. Prefill pushes thousands of prompt tokens through each weight it loads. Decode, for a single user, loads every weight to produce just one token.

Worked example

An 8-billion-parameter model in bfloat16 (2 bytes per parameter) has 16 GB of weights. The GPU reads memory at 2 TB/s and does 312 trillion FLOPs per second.

One decode step for one user: reading the weights takes 16 GB / 2 TB/s = 8 ms. The arithmetic, 2 × 8 billion = 16 billion FLOPs, takes 0.05 ms. The chip's arithmetic units sit idle almost the whole step, and one user gets about 125 tokens per second.

Prefill of a 2,048-token prompt: about 34 trillion FLOPs, which takes 108 ms. That's 0.05 ms per prompt token, against 8 ms per token when decoding for one user.

The roofline model from the previous page makes this precise. A step takes at least as long as its arithmetic at the chip's peak rate, and at least as long as its memory traffic at the chip's bandwidth. The ratio of the two, FLOPs per byte moved, says which limit wins. Prefill sits far above the chip's break-even ratio (the ridge), so it's compute-bound. Decode sits far below it, so it's memory-bound.[1]

That's why servers batch users together. Put 64 users in one decode step and the weights are still read once, so each extra user adds only their own arithmetic and their own cache reads. For decode with a batch of B sequences, each with T tokens of context:
tstep = max( F/P, M/W ) Fdecode = B·(2N + 4LHdT) Mdecode = bwN + B·T·k

In words: a step takes the longer of its arithmetic time and its memory time. Each sequence in the batch costs 2 FLOPs per parameter plus attention over its context. The memory side reads the weights once for the whole batch, plus every sequence's own cache of past tokens.

The 4LHdT term counts attention's work: scoring the new token against every cached token and mixing in their values. Prefill of T prompt tokens has F = 2NT plus attention that grows with T², and reads the weights once (bwN) plus the cache it writes. Batching raises decode's FLOPs per byte, but not without limit, because each added sequence also adds its own cache reads. For the 8B model at context 2,048 the ratio can never pass about 64 FLOPs per byte, well below this GPU's ridge of 156, so decode stays memory-bound at any batch size.
Decode step time and throughput against batch size
Model
Peak TFLOP/s
312
Bandwidth TB/s
2
Context
2048
Reset
8B, GQA (32 layers, 32 heads, 8 KV heads of 128) on a device with 312 TFLOP/s and 2 TB/s (ridge 156 FLOP/byte), bfloat16 weights and cache, context 2048. Prefill of 2048 tokens: 108.5 ms, 2082 FLOP/byte, compute-bound. One decode step at batch 1: 8.1 ms (1.0 FLOP/byte, memory-bound, 123 tokens/s); at batch 64: 16.6 ms (33 FLOP/byte, 3858 tokens/s). Weights alone are 14.90 GiB; each decode step reads them once plus 0.25 GiB of cache per sequence.

Each bar is one decode step at a batch size from 1 to 512; the right panel turns those steps into tokens per second. Try the model menu and the context slider. The numbers are roofline arithmetic, not measurements: they assume compute and memory traffic overlap perfectly and count only the model's own operations. The defaults are nanoGPT's 312 TFLOP/s bfloat16 figure for an A100 and 2 TB/s of bandwidth, roughly that class of chip; move the sliders to match your hardware.[2] For the 8B model at context 2,048, a batch-1 step takes 8.1 ms, almost all of it reading 16 GB of weights, for 123 tokens per second at 1 FLOP per byte against a ridge of 156. At batch 64 the step is 16.6 ms and the server makes 3,858 tokens per second; at 512 it makes 6,674, still memory-bound at 57 FLOPs per byte. Prefill of 2,048 tokens is one 108 ms step at 2,082 FLOPs per byte, compute-bound. The 7B model has 32 key-value heads, so each sequence reads four times as much cache and throughput flattens out sooner.
