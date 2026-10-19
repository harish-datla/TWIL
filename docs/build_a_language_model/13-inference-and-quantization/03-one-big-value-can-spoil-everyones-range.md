# One big value can spoil everyone's range

A layer's weight matrix has one row per output, and each row is called an output channel. Suppose one row holds a weight of 12 while another row's weights are all around 0.1. A single scale for the whole matrix (per-tensor) has to stretch to 12, and the small row's weights all round to within a step or two of zero. Giving each row its own scale (per-channel) fixes that, at the cost of storing one scale per row and needing a kernel that applies them.[3]

Worked example

Symmetric 8-bit codes run from −127 to 127. The matrix's largest weight is 12, so a shared scale is 12 / 127 = 0.0945.

The small row's 0.07 becomes round(0.07 / 0.0945) = round(0.74) = 1, which reads back as 0.0945. Its −0.03 becomes code 0 and reads back as 0.

With its own scale, 0.1 / 127 = 0.000787, the same 0.07 becomes round(88.9) = 89 and reads back as 0.07008. The error drops from 0.025 to 0.00008.
Quantize small and large rows
Large row coefficient
12
Code bits
Reset
Row 1 MSE per-tensor 8.773e-4 vs per-channel 4.185e-8. Shared scale 0.094488; row-1 scale 0.000787. Symmetric codes use ±(2^(bits−1)−1).

The left panel shows the small row [0.05, 0.07, −0.03, 0.1]: purple is the original, green uses the shared scale, yellow uses the row's own scale. The right panel shows each row's mean squared error. Drag the large coefficient up and the shared step grows, so the green bars snap to coarser levels; switch to 4 bits and the shared scale rounds the whole small row to zero. The large coefficient never changes the small row's original values, only how finely they can be stored.

This compares how well one small matrix survives the round trip. It shows per-channel scales losing less information here, not that a particular model keeps its accuracy.

Large language models have a harder version of the problem in their activations, the values passed from one layer to the next. Dettmers et al. found that in large transformers a few feature dimensions of the activations carry values far bigger than the rest. These are called outliers. Their method, LLM.int8(), quantizes most of each matrix multiply to 8 bits with a separate scale for every row of the activations and every column of the weights, and sends the outlier dimensions through a separate 16-bit multiply.[4]

SmoothQuant takes another route. It divides each activation input channel by a factor and multiplies the matching input channel of the weights by the same factor. The product doesn't change, but the activations' range gets smaller and easier to quantize, and some of the difficulty moves into the weights.[5]

Worked example

One activation channel reaches 60 while the others stay under 2. Its weights are about 0.02.

Pick a factor of 8 for that channel. The activation becomes 60 / 8 = 7.5 and the weight 0.02 × 8 = 0.16. Their product is unchanged: 60 × 0.02 = 7.5 × 0.16 = 1.2. The activation's largest value fell from 60 to 7.5, so a shared activation scale can be 8 times finer.

As a matrix identity, with s holding one positive factor per input channel:
XW = (X × diag(s)⁻¹) × (diag(s) × W)

In words: divide each input channel of the activations X by its factor and multiply the same input channel of the weights W by that factor. The exact product stays the same; only how hard each side is to quantize changes. The factors must line up with the dimension the multiply sums over.

Keep the two problems apart. The widget's per-channel scales were one per output row of the weights; activation outliers sit in input features of the activations. Those are different axes of different tensors. When someone says per-channel quantization fixed an error, ask which tensor and which axis.

Weights are fixed, so their ranges are known before deployment. Activations depend on the input. Static quantization fixes the activation ranges ahead of time by running representative inputs through the model and recording what values appear, a step called calibration. Dynamic quantization measures each activation's range while the model runs instead, and ONNX Runtime supports both.[3] Dynamic ranges cost extra work on every call; fixed ranges can clip inputs that look unlike the calibration set. Test whichever you pick on held-out inputs that resemble real traffic.
Check Your Understanding Q1 of 2

What does an affine integer code mean?
Its exact original floating-point value.
A probability between zero and one.
A value determined by its scale and zero point.
0/2 correct
