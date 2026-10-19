# Reuse keys and values during generation

A chatbot writing a 500-token reply generates one token at a time. Done naively, each new token reruns the whole model over the entire prompt and everything generated so far, recomputing hundreds of vectors that haven't changed. The KV cache stores those vectors instead.[6]

It works because of the causal mask. A token never attends to later tokens, so appending a new token doesn't change any earlier token's hidden states, keys or values, as long as positions keep the same numbering. For each new token: compute its query, key and value; append the key and value to the cache; attend over everything in the cache. This matches a full recomputation only when the weights are fixed and inference-time dropout (random zeroing used during training) is switched off.[6]

Worked example

Generate up to 7 tokens. Without a cache, step t reprocesses all t tokens: 1 + 2 + … + 7 = 28 token projections in total. With a cache, each step projects only the new token: 7 in total. The widget's readout counts the same thing.

The cache saves recomputation. It doesn't shrink attention: the new query still reads every cached key and value, so each step's attention work still grows with the length of the context. Processing the prompt, called prefill, can handle all the known tokens at once; the decoding steps after it commonly handle one new token per sequence.
Compare a full prefix with a cached query
KV heads in memory example
4
Step
Play
Reset
1 token; cumulative projected token rows full/cached 1/1; causal score pairs 1/1. Last-row error 0.0e+0. Hypothetical 8-layer cache (4 KV heads, dim 16, 2-byte entries): 2,048 bytes.

Click Step to append one token, or Play, and compare the two panels. Each token is a fixed two-number vector, used directly as its key and value with no learned projection. The left panel counts token rows projected so far: full recomputation reprocesses the whole prefix at every step and the cache only the new token, so after 7 tokens it's 28 against 7. The right panel shows the last token's output computed both ways; they match exactly. The KV-heads slider only changes the memory figure in the readout, which applies the cache formula below to a hypothetical 8-layer model with 16-wide heads and 2-byte entries.

The cache costs memory, and the amount is easy to compute. For an ordinary dense cache, the payload is:
bytes = 2 × layers × batch × KV heads × tokens × head width × bytes per entry

In words: every layer stores one key and one value (the factor 2) per head, per token, per sequence, each head width numbers long. Allocator overhead and metadata come on top.

Worked example

A model with 32 layers, 8 KV heads of width 128 and 16-bit entries: 2 × 32 × 8 × 128 × 2 = 131,072 bytes, or 128 KiB, per token. An 8,192-token conversation needs 1 GiB of cache. With 32 KV heads instead of 8, it needs 4 GiB.

Grouped-query attention gets that reduction by letting several query heads share one key/value head. It changes the model, so the model has to be trained or adapted that way; you can't reinterpret the cache of an ordinary multi-head model as a grouped one.[7]

The causal mask also needs care with a cache. A query in a cached chunk has an absolute position. When a 100-token prompt (positions 0 to 99) is cached and one new token arrives, that token is row 0 of a one-row chunk but sits at absolute position 100, so its mask must allow keys 0 through 100. A small triangular mask built for the chunk alone allows only key 0, the first cached key, and the code still runs. The code below makes the offset explicit.[6]

Full attention over N tokens scores every pair, so prefill does work that grows with N². Storing those scores as an N×N matrix is a choice, not a requirement: for N = 8,192 that's 67 million entries per head. FlashAttention computes exact attention in tiles and never writes the full matrix to the GPU's main memory, which cuts memory traffic. The equation and the number of score pairs stay the same; only the storage and scheduling of the work change.[8]
