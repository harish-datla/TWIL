# One small filter, reused everywhere

Say you want a network to tell whether a 32×32 color photo contains a cat. That photo is 32 × 32 × 3 = 3,072 numbers. A dense layer, the kind covered on the Neural Networks page, gives every input-output pair its own weight. A dense layer that outputs 16 numbers per pixel would need 3,072 × 16,384 ≈ 50 million weights, and it would have to learn what an ear looks like separately at every position. A convolutional layer learns one small grid of weights instead and slides it across the image, computing the same weighted sum at every position.[1]

That small grid is the kernel. At each position it sits on a patch of the image the same size as itself. Multiply each pixel in the patch by the kernel weight on top of it, add up the products, and that's one output number. Move one pixel to the right and do it again. Because the same weights are used everywhere, a kernel that responds to a vertical edge responds to it anywhere in the image. That's an assumption built into the layer, not knowledge: the weights still have to be learned from data before the kernel detects anything useful.[1]

Worked example

The image in the first widget is dark (0) on its left half and bright (1) on its right half. The vertical-edge kernel has the row (−1, 0, 1), repeated three times.

On a flat bright patch every row is (1, 1, 1): −1 + 0 + 1 = 0 per row, so the output is 0.

On the patch where the edge begins every row is (0, 0, 1): 0 + 0 + 1 = 1 per row, three rows, so the output is 3.

The kernel answers 0 on flat areas and 3 where dark meets bright. A bright-to-dark edge would give −3.

Written out, the output at one position is a sum over the kernel's offsets. Every convolution layer on this page computes this; later sections add channels and spacing options to it.
output[row, column] = Σu,v input[row + u, column + v] × kernel[u, v] + bias

In words: put the kernel's top-left corner on (row, column), multiply each kernel weight by the pixel under it, add everything up, then add the bias. The widget below uses one input grid, moves one pixel at a time, and has no bias.

Strictly, this is cross-correlation. The textbook convolution from signal processing flips the kernel left-to-right and top-to-bottom before sliding it. Deep learning libraries such as PyTorch skip the flip and still call the layer a convolution.[2] For learned kernels the difference doesn't matter, because training simply learns flipped weights. It matters when you copy a hand-written filter from a signal-processing source: the flip mirrors it, and for an edge kernel like the one above, mirroring changes the sign of every response.

The grid of outputs is a feature map: one number per position saying how strongly the patch there matched the kernel. It's not a list of detected objects. A large value means "this patch looks like the kernel", and a pattern running the other way can give a large negative value. An activation such as ReLU, which keeps positive values and zeroes out negative ones, then decides which responses pass to the next layer.

The widget has three hand-picked kernels so you can check the arithmetic yourself. The horizontal-edge kernel outputs zero everywhere on this image, because the brightness changes from column to column and never from row to row. The box average blurs. A trained network doesn't use hand-picked kernels: it learns its own through the loss and the backward pass, which section 06 covers.
Multiply one patch, then move it
Kernel
Step
Play
Reset
output[0,0] = 0.00 + 0.00 + 0.00 + 0.00 + 0.00 + 0.00 + 0.00 + 0.00 + 0.00 = 0.000

You're looking at the input with the current patch outlined, the 3×3 kernel, and the output map. Press Step or Play to slide the patch; the readout lists the nine products that add up to the highlighted output. It starts on the dark area, so every product is 0. The same nine weights produce every output in the map. Green is positive, pink is negative.

The weight count is the other payoff. Images usually have several numbers per pixel, one per channel: a color photo has three, red, green and blue. A layer with Cin input channels, Cout output channels and a K×K kernel has Cout × Cin × K² weights, plus one bias per output channel. The image's width and height don't appear anywhere in that count.[2]

Worked example

3 input channels (RGB), 16 output channels, 3×3 kernels: 16 × 3 × 9 = 432 weights plus 16 biases.

That's the same whether the image is 32×32 or 1024×1024. The dense layer at the top of this section needed about 50 million weights for the 32×32 case.
