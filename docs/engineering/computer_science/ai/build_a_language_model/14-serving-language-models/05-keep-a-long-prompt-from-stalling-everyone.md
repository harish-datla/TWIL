# Keep a long prompt from stalling everyone

Thirty-two users are watching their answers stream in when a new user pastes a 4,096-token document. With continuous batching, that prompt's prefill joins the next step, and a step that was a quick memory-bound decode becomes a long compute-bound prefill. Every streaming user's text freezes for that step.

Worked example

Same 8B model and GPU as the first section. A decode step for 32 users at context 2,048 takes 12.3 ms.

Add the whole 4,096-token prompt to one step and that step takes 226 ms: an 18× pause between two tokens for everyone already decoding.

Cap each step at 2,048 tokens: the 32 decode tokens plus 2,016 prompt tokens. The prompt spreads over three steps, and the longest takes 109 ms.

That cap is chunked prefill: split a long prompt across steps under a per-step token budget. vLLM turns it on by default and schedules the decode tokens before any prefill tokens. TensorRT-LLM calls it chunked context, with a default budget (max_num_tokens) of 8,192.[7][11] A smaller budget keeps the gap between tokens steady for users already decoding. A larger one gets the new user their first token sooner. vLLM's documentation suggests 2,048 when latency matters and more than 8,192 for throughput.[7]
Chunk a long prompt across decode steps
Model
Decoding sequences
32
Prompt tokens
Token budget per step
Reset
32 sequences decoding at context 2048 when a 4096-token prompt arrives, token budget 2048 per step. Decode-only step: 12.3 ms. The prompt is processed in 3 steps (2016 + 2016 + 64 tokens); the worst step the decoders sit through is 108.6 ms, 8.8× their usual gap. Time to the new request's first token: 229.4 ms.

Each bar is one step while the new prompt is being read; the green bars mark the usual decode-only step for comparison. Change the token budget, the number of decoding sequences and the prompt length. It's the same roofline arithmetic as the first widget, at the same A100-class defaults. With 32 sequences decoding at context 2,048, a normal step takes 12.3 ms. An unchunked 4,096-token prompt makes one step of 226 ms, 18 times the usual gap. A budget of 2,048 splits it into three steps of at most 109 ms, 9 times the gap; 512 gives nine steps of at most 27 ms, 2.2 times. The new request's first token arrives after 226 to 229 ms in every case, because the prefill does the same work however it's cut. What the budget buys is steady timing for the users already decoding, at the cost of a few extra milliseconds for the newcomer.
