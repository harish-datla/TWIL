# Subtract the zero points before you multiply

The point of integer codes is to run a layer's dot products (multiply matching entries, add them up) with cheap integer arithmetic instead of floating point. That works, but only if you subtract each vector's zero point from its codes before multiplying. Multiply the raw codes and the zero-point offsets get counted as if they were signal.[2]

Worked example

Both vectors use scale 0.1 and zero point 10. Input (0.5, −0.2) stores as codes (15, 8); weights (0.3, 0.4) store as (13, 14).

The real dot product: 0.5 × 0.3 + (−0.2) × 0.4 = 0.15 − 0.08 = 0.07.

Centered codes: (15 − 10)(13 − 10) + (8 − 10)(14 − 10) = 15 − 8 = 7. Times 0.1 × 0.1 gives 0.07.

Raw codes: 15 × 13 + 8 × 14 = 307. Times 0.01 gives 3.07, off by a factor of 44.
accumulator = Σ_i (input_code_i − input_zero) × (weight_code_i − weight_zero)
real_output ≈ input_scale × weight_scale × accumulator

In words: subtract the zero points, multiply matching codes, add the products in a wide integer, and convert to a real number once at the end by multiplying by both scales. The integer arithmetic is exact; the ≈ is the error from storing the inputs as codes in the first place.

This reference stops at a real number. A fully integer pipeline instead converts the accumulator straight into the next layer's integer codes, adds the bias in matching units, and clamps.

The running total, the accumulator, can get large, so its width matters. With 8-bit codes and any zero point, a centered code is at most 255 in size, so each product is at most 255² = 65,025 and a length-K dot product is at most K × 65,025 before bias. A signed 32-bit integer, int32, holds about 2.1 billion, so this worst case fits for K up to 33,025. A 4096-long dot product (at most 266 million) fits easily, but int32 isn't safe for any length without that check.
Compare corrected and raw-code products
Code bits
Integer product
Reset
Float dot 1.560000; corrected integer dot 1.552941; dequantized-vector dot 1.552941; raw-code dot 15.956863. Zero points 102/85; corrected accumulator 6732.

The left panel shows the integer codes for a four-number input and weight vector; the right compares the float dot product with the integer one. With zero points subtracted, the integer result is 1.553 against the float 1.56. It matches the dot product of the read-back vectors exactly, so the gap is quantization error. Switch to raw codes and it jumps to about 15.96, which is a different calculation. At 4 bits the corrected result drops to 1.333 because the steps are coarser.

A deployed integer-only network also has to handle the bias, stored in the same units as the product of the two scales, and the output, which needs its own scale and zero point. The framework and the hardware kernel must agree on where each conversion happens. Jacob et al. lay out the whole integer pipeline.[2]

A weight-only quantized model works differently. Its kernel can unpack the 4- or 8-bit weights back to floats on the fly and do floating-point math, so the activations never leave floating point. That's a different arithmetic path from an all-int8 matrix multiply. A model file's bit count doesn't tell you which path runs, how much unpacking it needs, or whether the kernel got any faster.
