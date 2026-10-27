# Stride, padding and dilation set the output size

Before you stack layers you need to know what size comes out of each one. A 3×3 kernel on a 7×7 image only fits in 5×5 positions, so the image shrinks a little at every layer. Many networks also shrink it on purpose, so later layers have fewer positions to compute. Three settings control the output size.[3]

    Stride is how far the kernel moves between placements. Stride 2 skips every other position, which roughly halves each side of the output.
    Padding adds extra values, usually zeros, around the border so the kernel can hang partly off the edge.
    Dilation spreads the kernel's samples apart. With dilation 2, a 3-wide kernel covers 5 pixels but still has only 3 weights.

Worked example

One row of 7 pixels, a 3-wide kernel, padding 1, stride 2. Padding makes the row 9 wide, positions −1 to 7.

The kernel's left edge can sit at −1, 1, 3 and 5. At 7 it would run off the end. That's 4 outputs.

Now set dilation to 2. The kernel covers 5 positions, so its left edge can only sit at −1, 1 and 3: 3 outputs, still from 3 weights each.

The general rule counts the same thing: how many stride-sized steps the kernel can take across the padded row.
K_eff = dilation × (K − 1) + 1
O = floor((I + 2 × padding − K_eff) / stride) + 1

In words: first work out how wide the kernel really is once the gaps are included. Then take the padded input width, subtract the kernel width, divide by the stride and round down: that's how many steps fit. Add 1 for the starting position. For the example, (7 + 2 − 3) / 2 + 1 = 4, and with dilation 2, (7 + 2 − 5) / 2 + 1 = 3. The formula assumes the same padding on both sides.

Zero padding is a guess about what lies outside the image, and the kernel treats it like real data. At the border it sees real pixels next to zeros, which can look like an edge that isn't in the scene. Reflection or circular padding make different guesses and give different numbers. A common recipe keeps the size unchanged: with an odd kernel size K, stride 1 and dilation 1, pad by (K − 1)/2 on each side, which is 1 for a 3×3 kernel. With stride above 1, definitions of "same size" differ, and some of them pad one side more than the other.[3]
Count legal kernel placements
Stride
2
Padding
1
Dilation
1
Step
Play
Reset
effective kernel 3; output 4×4; selected output uses 4 real samples + 5 padded zeros

You're looking at a 7×7 input and the output grid it produces. Yellow input cells are the pixels the kernel actually reads for the highlighted output; positions in the padding count as zeros and aren't drawn. Press Step to move to the next output, and raise dilation to open gaps between the samples. At the defaults (stride 2, padding 1, dilation 1) the output is 4×4, and the first output reads 4 real pixels and 5 padded zeros.

Image generators and segmentation networks often need to go the other way, from a small map to a bigger one. A common tool is transposed convolution: each input value stamps a scaled copy of the kernel onto a larger output, and overlapping stamps add up. It's the same arithmetic the backward pass uses to send gradients back to input pixels. Despite the name it's not an inverse. A stride-2 layer with one input and one output channel produces roughly a quarter as many numbers as it reads, so information is gone and no layer can bring it back. The output_padding option some libraries offer only picks between output sizes that would otherwise be ambiguous; it doesn't restore lost values.[3]
