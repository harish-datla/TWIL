# Where the memory goes

An 80 GB GPU has to hold the model's weights and the working memory of every conversation in flight. The question a serving engine keeps asking is how many conversations fit. Weights take 2 bytes per parameter in bfloat16: 16 GB for an 8B model, and 140 GB for a 70B model, which is why models that size are split across several devices.[3]

The rest goes mostly to the KV cache. Attention compares each new token against every earlier one using two vectors per earlier token per layer, a key and a value. Recomputing them for every new token would be wasteful, so the server keeps them. Unlike the weights, this cache grows with every token of every active conversation.

Its size per token depends on how many key-value heads each layer has. Grouped-query attention lets several query heads share one key-value head. Llama 3 8B has 32 query heads sharing 8 key-value heads, four to each, so it stores a quarter of the cache per token that a same-sized model with one key-value head per query head would.[4][5][6]

Worked example

Llama 3 8B: 32 layers, 8 key-value heads of width 128, 2 bytes per value. Per token: 2 (key and value) × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KiB.

A 4,096-token conversation: 128 KiB × 4,096 = 0.5 GiB.

On an 80 GiB device, the weights take 14.9 GiB, leaving 65.1 GiB, and 65.1 / 0.5 = 130 conversations at full length.

A 7B model of the same shape but with 32 key-value heads, one per query head, needs 512 KiB per token and 2 GiB per conversation. Beside its 12.5 GiB of weights, only 33 fit.
k = 2 · L · Hkv · d · bc sequences ≈ (Mdevice − bwN) / (k·T)

In words: each token costs a key and a value, in every layer, for every key-value head, at the cache's precision. Subtract the weights from device memory and divide what's left by one full conversation's cache to get how many conversations fit.

That count is an upper bound. Activations, the framework's workspace, fragmentation and CUDA graphs all come out of the same memory. That's why vLLM reserves a configured fraction of memory for the cache and preempts requests when it runs out.[7]
Size the cache and count the sequences that fit
Model
Cache bytes
Context
4096
Device GiB
Reset
8B, GQA (32 layers, 32 heads, 8 KV heads of 128): 1,31,072 bytes of cache per token at 2 bytes per value (128 KiB), 0.50 GiB per sequence at context 4096. Weights in bfloat16: 14.90 GiB of 80 GiB, leaving 65.10 GiB for cache: room for 130 concurrent sequences at full context, before activations and framework overhead.

The left panel stacks weights (purple) and cache (green) for 1, 4, 16 and 64 sequences against the device's memory; the right counts how many full-length sequences fit at each context length. Change the model, cache precision and device size. For the 8B grouped-query model in bfloat16, the cache is 128 KiB per token: half a GiB per 4,096-token sequence, and 16 GiB for 32 of them, more than the weights. On an 80 GiB device that leaves room for 130 sequences at context 4,096 and 16 at 32,768. The 7B multi-head model's cache is 512 KiB per token, so only 33 fit at 4,096. An 8-bit cache doubles every count. llama.cpp offers K and V cache types down to 4-bit, and Transformers ships a quantized cache for the same reason.[8][9] Sliding-window layers are the other lever: their cache stops growing at the window length, whatever the context.[9]
