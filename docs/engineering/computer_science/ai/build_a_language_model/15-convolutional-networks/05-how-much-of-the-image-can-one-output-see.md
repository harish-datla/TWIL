# How much of the image can one output see?

A 3×3 kernel sees a 3×3 patch, far too small to recognize a face. CNNs get around that by stacking layers. A second-layer output combines a 3×3 block of first-layer outputs, each of which saw its own 3×3 patch, so it depends on a 5×5 patch of the input. The set of input pixels one output can depend on is its receptive field, and its size tells you whether a network can see enough context for the task.[6]

Downsampling makes it grow faster. After a stride-2 layer, neighboring outputs sit 2 input pixels apart, so the next 3-wide kernel reaches 4 more input pixels instead of 2. Bookkeeping needs two numbers per layer: the span, how many input positions one output covers, and the jump, how many input pixels apart neighboring outputs are.

Worked example

Three layers with 3-wide kernels and stride 1, in one dimension. The span starts at 1 and each layer adds 2: 1 + 2 + 2 + 2 = 7 input positions.

Give the first layer stride 2. It still adds 2, because its inputs are 1 pixel apart, but it sets the jump to 2. The next two layers add 2 × 2 = 4 each: 1 + 2 + 4 + 4 = 11. Neighboring final outputs are 2 input positions apart.

The general recurrence walks from the input up through the layers, updating both numbers at each one.
span_next = span + (K − 1) × dilation × jump
jump_next = jump × stride

In words: each layer widens the span by its kernel's reach, which is (K − 1) × dilation steps, each step jump input pixels wide. A stride multiplies the jump for every layer after it. Start with span 1 and jump 1 at the input. This works for a plain chain of layers; a network with branches needs the count for each path.
Trace dependencies back to input positions
Layers
3
First-layer stride
2
Dilation
1
Reset
Final output index 1; 3 layers; first stride 2; dilation 1. Input positions [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]. Bounding span 11, distinct contributing positions 11, output spacing 2.

You're looking at the positions that feed one final output (index 1), traced back one layer per row down to the input. Change the number of layers, the first layer's stride, or the dilation. At the defaults (3 layers, first stride 2, dilation 1) the input row covers positions 2 to 12: span 11, all 11 positions used. Set dilation to 2 and the span grows to 21 while only 11 distinct positions are read, because the gaps leave holes. Each row uses that layer's own position numbers, and there's no padding.

The receptive field says what can affect an output, not how much. A zero weight cuts a path completely; a ReLU that outputs zero for this input cuts it for this input only. Dilation can leave holes, so a large span may contain positions no kernel ever reads. When judging coverage, count the positions actually sampled.[6]

Networks with branches need one more check. A residual connection adds a stack of layers' output back onto that stack's own input, written F(x) + x by He et al.[7] The addition is entry by entry, so both tensors need the same shape, and entry (row, column) in each must refer to the same place in the image. When the layers change the channel count or downsample, He et al. pass the shortcut through a learned projection, F(x) + Wsx, so the shapes match.
Check Your Understanding Q1 of 2

A dilated dependency span is 13 positions. Must all 13 contribute?
Yes; the span is a dense input window.
Only if the activation is ReLU.
No; dilation and stride can leave holes.
0/2 correct
