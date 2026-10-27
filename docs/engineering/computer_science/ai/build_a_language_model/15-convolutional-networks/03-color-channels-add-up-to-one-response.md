# Color channels add up to one response

A color photo isn't one grid but three: red, green and blue. After the first layer the channels are whatever the previous layer's kernels computed, and there are usually more of them; ResNet's first layer outputs 64.[7] A kernel looking for a red-to-green edge has to see red and green together, so an ordinary convolution doesn't filter each channel on its own. Each output channel has one small kernel per input channel. It runs each kernel over its channel, adds the results into a single map, adds one bias, and only then applies the activation.[2]

Worked example

The next widget has two input channels and one output channel, so it has 1 × 2 × 9 = 18 weights and one bias (set to 0).

At the top-left output, channel 0's kernel contributes 1.50 and channel 1's kernel contributes −1.50.

The output is 1.50 + (−1.50) + 0 = 0.

With channels, the formula from section 01 gains one more sum, over the input channels.
output[o, row, column] = bias[o] + Σc Σu,v
input[c, row + u, column + v] × weight[o, c, u, v]

In words: for output channel o, run the kernel weight[o, c] over input channel c, do that for every input channel, then add all the results and one bias. Each output channel has its own complete set of kernels, one per input channel.

In the widget, channel 0 brightens from left to right and channel 1 brightens from top to bottom. Channel 0's kernel responds to left-to-right change; channel 1's kernel subtracts its bottom row from its top row, so a top-to-bottom brightening gives a negative number. Both ramps are perfectly even, so every position gets the same two numbers. The gain slider scales channel 1's kernel. At gain 1 the contributions cancel and the summed map is zero everywhere. That's a legitimate answer, not a bug: the layer combines evidence from both channels before any activation sees it.
Two channels, one output map
Second-channel gain
1
Reset
Two input channels → one output channel. At [0,0]: 1.50 + -1.50 = 0.00. 18 weights, one bias (bias=0).

You're looking at what each of the two input channels contributes to the one output channel, and their sum. Drag the gain to scale channel 1's kernel. At the default gain of 1 the two contributions cancel and the sum is 0 everywhere; at 0 the sum equals channel 0's map. The left two maps are parts of one output, not separate output channels. All three share one color scale, and the bias is 0.

Two variations are common. A 1×1 convolution has a kernel one pixel wide, so it never looks at neighbors. It only mixes channels, separately at each pixel, like a small dense layer applied to every pixel's list of channel values. That makes it a cheap way to raise or lower the channel count. Two 1×1 layers with nothing between them are just two matrix products and could be merged into one; put an activation between them and they can't.

A depthwise convolution goes the other way: each input channel gets its own spatial kernel (or several), and channels are never mixed. Following it with a 1×1 layer splits one layer's job in two, one step for space and one for channels. Depthwise is the extreme case of grouped convolution, where channels are split into groups that never connect, and the weight count becomes Cout × (Cin / groups) × K².[2]

Worked example

64 input channels to 64 output channels with 3×3 kernels. An ordinary layer: 64 × 64 × 9 = 36,864 weights.

Depthwise 3×3 (groups = 64): 64 × 1 × 9 = 576 weights. Then a 1×1 layer: 64 × 64 × 1 = 4,096. Together: 4,672, about an eighth of the ordinary layer.
Check Your Understanding Q1 of 2

Why are the two channel maps summed before ReLU?
Each channel must be classified separately.
The sum replaces all trainable parameters.
They are contributions to one output channel.
0/2 correct
