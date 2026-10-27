# Store each number as a small integer

A 7-billion-parameter model stored in 16-bit floats needs 14 GB for its weights alone, more than many consumer GPUs have. Store each weight as an 8-bit integer and it's 7 GB; at 4 bits, about 3.5 GB. Replacing real numbers with a small set of integer codes like this is quantization. This section shows how the codes work and what they cost in accuracy.

The scheme here, affine quantization, picks a range of real values to cover, splits it into equal steps, and stores the number of the step each value is nearest to. Two extra numbers turn a code back into a real value: the scale, which is the size of one step, and the zero point, which is the code that stands for real 0. A code on its own means nothing: the same codes with a different scale are different numbers.[2]

Worked example

Cover −3 to 3 with 8-bit codes 0 to 255. Scale = 6 / 255 = 0.0235. Zero point = round(3 / 0.0235) = round(127.5) = 128.

Store 1.1: 1.1 / 0.0235 = 46.75, which rounds to 47; add the zero point to get code 175.

Read it back: 0.0235 × (175 − 128) = 1.106. The error is 0.006, under half a step (0.012).

Now narrow the range to −1 to 1 and store 2.7. It lands past the top, so it's clamped to code 255 and reads back as 0.996. The error is 1.7, and all of it comes from clipping.

Written out, storing a value and reading it back are:
integer = clamp(round(real / scale) + zero_point, q_min, q_max)
reconstructed = scale × (integer − zero_point)

In words: to store, divide by the step size, round to a whole number, shift by the zero point, and clamp into the code range. To read back, undo the shift and multiply by the step size. Only the rounding and the clamp lose information.

With b bits there are 2b codes, so a common choice sets scale = (high − low) / (2b − 1). The range is widened if needed so it includes 0, and the zero point is rounded to an integer, so real 0 gets an exact code; Jacob et al. require this because efficient kernels often pad arrays with zeros.[2] Rounding the zero point can shift the ends of the range slightly.

That leaves two kinds of error. Rounding error, for values inside the range, is at most about half a step. Clipping error, for values outside it, can be as big as the value itself. Picking the range trades one for the other: a wide range never clips but spreads the codes farther apart, while a narrow one gives fine steps and chops off the big values. The widget lets you move that trade-off on eight fixed values. Its mean squared error describes only this array.
Separate rounding error from clipping
Range half-width
3
Code bits
Reset
8-bit affine codes; scale 0.023529, zero point 128; MSE 0.000048; 0 inputs outside representable endpoints. Codes [34, 77, 111, 128, 141, 175, 205, 243].

Purple bars are eight fixed values from −2.2 to 2.7; green bars are what comes back after storing them as codes, and the right panel shows the error for each. Drag the range half-width below 2.7 and the outer values get clipped, with large errors. Switch to 4 bits and every value lands on a coarser grid. Bar 3 is exactly 0 and always comes back exact, because its code is the zero point.
Exact halves: which way to round

A simpler variant, symmetric quantization, makes the range run from −max to +max so the zero point is just 0. There's one less number to track and the multiply gets simpler. But if the values are lopsided, say all between −0.1 and 3, almost half the codes cover values that never occur. Which scheme works best depends on the tensors and on what the kernels support as much as on the bit count.
