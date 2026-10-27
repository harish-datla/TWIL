# Memory, and buying it back with recomputation

A GPU that holds a model's weights comfortably can still run out of memory during training. One common reason is the values the backward pass needs. Forward mode stores nothing: each tangent is used up as soon as it's produced, so its memory doesn't depend on how deep the computation is. Reverse mode must keep every forward value a local derivative needs, such as both inputs of every multiply and the cosine for every sin, until the backward pass reaches it. So its memory grows with depth.[2] In a deep network those saved values are the activations, and at large batch sizes or long sequences they can take more memory than the parameters themselves.

Activation checkpointing trades compute for that memory. Cut the network into segments. The forward pass keeps only the input to each segment and throws away everything inside it. When the backward pass reaches a segment, it reruns that segment's forward computation from the saved input to get the values back.[4][5]

Worked example

A chain of 100 equal layers, counting each layer's activation as one unit. Without checkpointing, all 100 are alive when the backward pass starts.

With segments of 10: the forward pass keeps 10 segment inputs. The backward pass rebuilds one segment at a time, so at most 10 more are alive at once. Peak: 10 + 10 = 20. Cost: the 90 layers that aren't segment inputs run forward a second time, 90% extra forward work.

The formula below gives that peak for any chain length and segment length, which tells you where to cut.
peak ≈ L / s + s

In words: the peak is the number of saved segment inputs plus the size of the one segment being rebuilt. Short segments mean many saved inputs; long segments mean a big rebuild. The two balance at s = √L, where the peak is about 2√L instead of L.

The price is one extra forward evaluation of every recomputed layer, close to one extra forward pass in total.
Choose a segment length for a chain of layers
Layers L
64
Segment length s
8
Reset
64 layers, segments of 8: keep 8 boundary activations, recompute 56 layers during the backward pass (88% extra forward work). Peak live activations 16 versus 64 without checkpointing. The sweep's minimum peak is 16 at segment 8; √64 ≈ 8.0.

Drag the number of layers L and the segment length s. The left panel plots the peak number of live activations for every segment length, with your choice marked; the bars compare what's stored, recomputed and live at the peak, with and without checkpointing. The count treats every layer's activation as one unit. Without checkpointing all 64 activations are live at the start of the backward pass. With segments of 8, the forward pass stores 8 boundaries and the backward pass recomputes 56 layers, 88% extra forward work, for a peak of 16. The sweep bottoms out at s = 8 = √64. The count assumes each layer is recomputed once, from the nearest stored boundary; PyTorch's checkpoint_sequential instead reruns every segment except the last in full, which costs nearly one extra forward pass. Real policies are less uniform: JAX lets a checkpoint policy save compute-heavy results such as matrix products and recompute cheap ones.[5]
