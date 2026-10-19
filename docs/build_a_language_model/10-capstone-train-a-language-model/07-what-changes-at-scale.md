# What changes at scale

How far is this toy from a real model? Mostly, the constants. Change them and the same code becomes nanoGPT's Shakespeare model, then GPT-2. What grows is the parameter count, the compute and the memory, and each one follows from the model's shape with a little arithmetic.

nanoGPT's Shakespeare run uses 6 blocks of 6 heads at width 384, context 256, batch 64 and dropout 0.2, for 5,000 iterations at a peak rate of 1e-3. That's about 10.7 million parameters, 3 minutes on one A100, and a best validation loss of 1.4697 nats, 2.12 bits per character.[11][13] GPT-2 124M is 12 blocks of 12 heads at width 768, context 1,024 and a 50,257-token BPE vocabulary. nanoGPT trains it on OpenWebText for 600,000 iterations of 491,520 tokens, on eight A100s for about four days, to a loss near 2.85 nats per token.[13][4][14]
Where GPT-2 came from

Parameters. Each block holds 12C² weights in its attention and MLP matrices, where C is the width: 4C² in attention (query, key, value and output projections) and 8C² in the MLP. So doubling the width quadruples every block, while doubling the depth only doubles the total. The embedding table adds V·C, which is large for a 50,257-token vocabulary.

Worked example

GPT-2 124M: 12 × 768² = 7,077,888 weights per block, × 12 blocks = 84.9 million. Token embedding 50,257 × 768 = 38.6 million. Position embedding 1,024 × 768 = 0.8 million. Biases and LayerNorm parameters: 121,344.

Total: 124,439,808, with the token embedding 31% of it.

Compute. Training costs close to 6 floating-point operations (FLOPs) per parameter per token: 2 for the forward pass, one multiply and one add per weight, and 4 for the backward pass. Attention's scores and weighted sums add 12·L·H·Q·T per token, for L blocks of H heads of width Q over context T. nanoGPT includes that term when it estimates utilization, the share of the hardware's peak speed a run actually uses, following the PaLM paper.[7]

Worked example

GPT-2 124M, counting parameters without the position table as nanoGPT does (123.7 million): 6 × 123.7M = 7.42 × 10⁸, plus 12 × 12 × 12 × 64 × 1,024 = 1.13 × 10⁸ for attention, gives 8.55 × 10⁸ FLOPs per token.

10¹¹ tokens: 8.55 × 10¹⁹ FLOPs. Eight A100s at 312 TFLOP/s each, 40% utilized, do about 10¹⁵ FLOPs per second, so the run takes about 23.8 hours.

Memory. Training with Adam in mixed precision holds about 16 bytes per parameter: 2 for the bfloat16 weights, 2 for their gradients, and 12 for a 32-bit master copy of the weights and Adam's two running averages. That 12 is why DeepSpeed's ZeRO splits exactly those optimizer states across devices, and it reports 18 GB of Adam state for a 1.5-billion-parameter GPT-2.[15]

Worked example

GPT-2 124M: 124.4 million × 16 bytes = 2.0 GB, which the widget shows as 1.85 GiB (1 GiB = 2³⁰ bytes). GPT-2 XL, 1.56 billion parameters: 24.9 GB, of which the 12 bytes of master weights and Adam state are 18.7 GB, in line with DeepSpeed's 18 GB.

Data. The Chinchilla fit from the Training Deep Networks page puts compute-optimal training at about 20 tokens per parameter, which is 2.5 billion tokens for GPT-2 124M.[16] This page's model sees 900 × 256 = 230,400 tokens, about 15 per parameter, but they come from only 14,361 training characters, so it reads its training text about 16 times over.
Count parameters, FLOPs, time and memory for a GPT shape
Model
Tokens 10ᵏ, k
11
Hardware
Utilization %
40
Reset
GPT-2 124M: 12 layers, width 768, context 1024, vocabulary 50257 → 12,44,39,808 parameters (12,36,53,376 excluding positions; embeddings 31.0%). Training on 1e+11 tokens (809 per parameter): 8.55e+19 FLOPs by 6N + 12·L·H·Q·T per token (6ND alone: 7.42e+19), 23.8 h on eight A100s at 40% utilization. Weights, gradients and Adam state in mixed precision: 16 bytes per parameter = 1.85 GiB.

Pick a model shape, a token count, hardware and a utilization. The left bars split the parameters into groups; the right bars show the bytes each parameter costs during mixed-precision training. Parameter counts are exact for the GPT-2 shapes, with biases: 124,439,808 for GPT-2 124M and 1,557,611,200 for XL, and 10,745,088 for nanoGPT's bias-free Shakespeare model, which nanoGPT reports as 10.65M after excluding the position table.[7] For GPT-2 124M the token embedding is 31% of the total; for XL it's 5%. At 10¹¹ tokens, forty percent utilization of eight A100s puts the 124M run at about a day (23.8 hours) by the 6N + 12LHQT count. That's consistent with nanoGPT's four-day figure, which covers about 3 × 10¹¹ tokens at a lower realized utilization.[13] The memory panel is arithmetic, not a measurement: activations, the KV cache and framework overhead come on top.

The data changes too. Character-level Shakespeare is one author writing in one style. GPT-2's WebText was filtered web text. Later open corpora mix many sources with explicit weights: the Pile combines 22, with Common Crawl at 18%, PubMed Central at 14%, books at 12%, and GitHub, arXiv and more, to about 1,254 GiB after repeating some sources for up to three passes.[17] At that size, removing duplicates, filtering out junk and keeping the validation text out of the training text become engineering problems of their own, which the Evaluation page covers.
Check Your Understanding Q1 of 2

GPT-2 124M has a 50,257-token vocabulary and width 768. What fraction of its parameters is the token embedding?
Under 5%.
About 31%.
Over half.
0/2 correct
