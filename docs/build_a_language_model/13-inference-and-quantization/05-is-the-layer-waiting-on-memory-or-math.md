# Is the layer waiting on memory or on math?

When a model generates text for one user, each new token runs every layer on a single input vector. Each layer reads its whole weight matrix from memory but does only one multiply and one add per weight. Whether that's slow because of arithmetic or because of memory decides what will speed it up. The roofline model answers the question with two numbers about the hardware: its peak arithmetic rate and its memory bandwidth.[6]

What decides between them is arithmetic intensity, the number of operations done per byte moved. At low intensity, memory can't feed the arithmetic units fast enough. At high intensity, arithmetic is the limit. Say which memory you're counting bytes from: an estimate for on-chip cache and one for device memory are different models.

Worked example

A made-up chip: 10 TFLOP/s peak, 200 GB/s memory bandwidth. A 4096 × 4096 layer with 2-byte weights, one input row.

Work: 2 × 4096² ≈ 33.6 million operations. Traffic: 33.6 MB of weights, plus 16 KB of input and output. Intensity ≈ 1 operation per byte.

Memory time: 33.6 MB / 200 GB/s = 0.168 ms. Arithmetic time: 33.6 million / 10 trillion = 0.0034 ms. The layer is memory-bound by about 50×.

To keep the arithmetic busy, this chip needs 10 T / 200 G = 50 operations per byte. A batch of 64 rows reaches 62, and the layer becomes compute-bound.
throughput ≤ min(peak_ops_per_second, bandwidth × intensity)
time ≥ max(operation_count / peak, byte_count / bandwidth)

In words: a layer can't go faster than the arithmetic units allow or faster than memory can deliver its bytes, so its time is at least the larger of the two. Real kernels land above this bound: launch overhead, dependencies between steps, conversion work and imperfect use of the hardware all add time.

Take a weight matrix with M rows and K columns applied to B input rows. The work is 2BMK operations (each multiply and each add counts as one). The weights are read once for the whole batch, so a bigger batch does more work per weight byte. The widget's model also counts reading the inputs and writing the outputs once each, and ignores every other tensor.
Change batch reuse and weight storage
Batch rows
1
Square layer width
Bytes per weight
Hypothetical peak TFLOP/s
10
Hypothetical GB/s
200
Reset
Hypothetical bound: memory; intensity 1.00 ops/byte; traffic 32.02 MiB; optimistic lower time bound 0.1679 ms. One 4096×4096 layer, 1 input rows; no measured latency.

The left plot shows the best possible throughput of one square layer on a made-up chip as the batch grows; the dot marks the current batch. Drag the batch from 1 toward 64: throughput climbs in a nearly straight line while memory is the limit, then flattens at the arithmetic peak. Switch to 1-byte weights and the weight traffic halves, so the memory-bound part doubles. Inputs and outputs stay at 2 bytes per element, and the peak doesn't change with the byte setting. It's an optimistic bound for one layer, not a GPU benchmark or a promised int8 speedup.

So batching helps: serve 32 users at once and each weight read does 32 users' work. The catch is waiting, because a request may sit in a queue while a batch forms. Throughput across all users, time to the first token, and each request's latency are separate measurements, and this bound covers only the execution of one layer. Queueing time comes on top.

A transformer generating text also keeps a cache. For every earlier token, each layer stores that token's attention keys and values, vectors it computed once and reuses for every later token instead of recomputing. This key-value (KV) cache takes 2 × layers × batch × KV heads × tokens × head width × bytes per entry, as the transformer page derives.[7]

Worked example

32 layers, 8 KV heads of width 128, 2-byte entries, one sequence: 2 × 32 × 8 × 128 × 2 = 131,072 bytes, or 128 KiB per token. A 4096-token conversation holds 512 MiB of cache.

Quantizing the weights doesn't shrink that cache. It grows with every token, and with full attention each new token reads all of it.
Check Your Understanding Q1 of 2

Does storing weights in one byte instead of two guarantee twice the inference speed?
Yes; runtime is always proportional to model file size.
Only if the softmax temperature is doubled.
No; the executed kernels and actual bottlenecks determine speed.
0/2 correct
