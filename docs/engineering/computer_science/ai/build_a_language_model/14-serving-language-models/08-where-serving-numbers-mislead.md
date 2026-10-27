# Where serving numbers mislead
Throughput and latency are bought from each other

Every lever on this page trades one for the other. A larger batch raises tokens per second and lengthens every user's step. A larger prefill budget gets new users their first token sooner and makes everyone already decoding wait longer between tokens. Speculative decoding speeds up a lightly loaded server and can slow down a saturated one. Report tokens per second together with time to first token and the spread of inter-token latency at a stated number of concurrent users; a throughput figure alone can't be compared with anything.

Roofline estimates are lower bounds. The first widget's step times assume compute and memory overlap perfectly and leave out kernel launches, synchronization, sampling and tokenization. Those costs matter most at small batches, where the step itself is short, so measured steps there can be well above the estimate.

Streaming changes what latency means. TGI's documentation uses a 1,000-token answer at 100 tokens per second: it takes ten seconds either way, but with streaming the first words appear at once, and the user can stop a generation that's going wrong.[17] Users feel time to first token and the gap between tokens; batch jobs feel end-to-end time.

Prefix caching can leak information. If tenants share a cache, a request can tell from its own speed whether someone else recently sent the same prefix, because a cached prefix skips prefill. vLLM's per-request cache salt limits reuse to requests that share the salt, so each trust group gets its own cache.[13]
