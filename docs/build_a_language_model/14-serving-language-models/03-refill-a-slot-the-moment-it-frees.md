# Refill a slot the moment it frees

Requests arrive at random times and want answers of very different lengths: one user asks for a yes or no, another for a 500-word essay. The simplest server groups requests into a fixed batch and runs it until every one is done. Short answers finish early and leave their slots idle, while new arrivals wait outside for the longest answer in the batch to end.

Continuous batching decides who runs one step at a time instead. When a sequence emits its end token, its slot is handed to the next request in the queue on the next step, and the newcomer's prefill runs in the same step as everyone else's decode.[10][11]

Worked example

Two slots. Three requests are waiting: A wants 10 tokens, B and C want 2 each.

Fixed batch: A and B run in steps 1 to 10. B is done after step 2, and its slot sits idle for 8 steps. C starts only when the whole batch ends, runs in steps 11 and 12, and gets its answer after 12 steps.

Continuous: C takes B's slot at step 3 and is done after 4 steps. Everything finishes by step 10 instead of 12.

TensorRT-LLM calls the same idea in-flight batching, and its documentation describes exactly this mix of prompt-reading and generating sequences in one batch. llama.cpp's server runs it by default across its parallel slots.[11][8]
Static against continuous batching on one request stream
Slots
Mean arrival gap (steps)
3
Seed
1
Reset
40 requests, one every 3 steps on average, outputs of 8 to 64 tokens, 8 slots. Static batching: 325 steps to drain, mean latency 123.1 steps (mean wait before admission 88.5), slot utilization 53%, 4.26 tokens per step. Continuous: 211 steps, mean latency 59.6 (wait 25.0), utilization 82%, 6.55 tokens per step.

Each row is a slot and each column a step; a colored cell is a request using that slot, and a blank one is idle. Compare the two timelines, then change the slot count, arrival gap and seed. The same 40 requests, with answers of 8 to 64 tokens, arrive every 3 steps on average. Each step runs one decode for every active sequence, and joining costs nothing beyond the step a request joins in. With 8 slots, static batching drains the stream in 325 steps with a mean latency of 123 steps, 89 of them spent waiting for the current batch to finish, at 53% slot utilization. Continuous batching drains it in 211 steps at a mean latency of 60, with 25 steps of waiting and 82% utilization. The gap closes once slots outnumber arrivals: at 16 slots continuous batching never queues a request. The timelines count steps, not seconds, so they hide one effect: a step with more active sequences takes longer, as the first widget showed.
Check Your Understanding Q1 of 2

On the roofline, a decode step for 8 sequences takes about as long as a step for 1. Why?
The weights are read from memory once per step regardless of batch size, and reading them dominates the step.
The GPU runs the 8 sequences on 8 separate cores in parallel.
The KV cache is shared between the sequences.
0/2 correct
