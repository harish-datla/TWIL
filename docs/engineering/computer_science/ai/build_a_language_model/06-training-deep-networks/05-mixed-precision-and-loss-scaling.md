# Mixed precision and loss scaling

A float32 number takes 4 bytes and a 16-bit number takes 2. On GPUs with tensor cores, 16-bit matrix multiplies run several times faster than 32-bit ones and move half the bytes. The catch is that 16 bits can't hold every number a training run produces. Some gradients round to zero and those weights stop learning; others overflow to infinity and the loss becomes NaN.

Mixed precision training runs the heavy operations, mainly matrix multiplies, in a 16-bit format and keeps a 32-bit master copy of the weights for the optimizer update. In PyTorch a feature called autocast picks the precision for each operation.[12] Which 16-bit format you choose decides what can go wrong.

float16 spends 5 bits on the exponent, which sets its range, and 10 on the fraction, which sets its precision. Its largest finite value is 65,504. Its smallest normal number is 2⁻¹⁴ ≈ 6.1×10⁻⁵, and below that, subnormal numbers reach down to 2⁻²⁴ ≈ 6×10⁻⁸ with less precision. Gradients below about half of that round to zero, and the parameters they belong to get no update at all.

The fix is loss scaling. Multiply the loss by a large factor before the backward pass; by the chain rule, every gradient gets multiplied by the same factor, which lifts the small ones into range. Then divide the gradients by the factor before the optimizer step.[12]

Worked example

A gradient of 10⁻⁸ rounds to 0 in float16.

Multiply the loss by 2¹⁶ = 65,536 and that gradient becomes 6.55×10⁻⁴, well inside float16's normal range. Dividing it back in float32 recovers 1.0×10⁻⁸ to within 0.03%.

The same scale turns a gradient of 2 into 131,072, which is above 65,504 and overflows to infinity.

So no single scale suits a whole run. PyTorch's GradScaler adjusts it as training goes: it starts at 2¹⁶, and whenever the gradients contain an inf or NaN it skips that step and halves the scale. After 2,000 clean steps in a row it doubles the scale again.[13]

bfloat16 makes the opposite trade. It keeps float32's 8 exponent bits, so it has float32's range and needs no loss scaling, and keeps only 7 fraction bits, so every value is rounded more coarsely. nanoGPT trains in bfloat16 when the GPU supports it and falls back to float16 with a GradScaler otherwise.[9] The bit layouts and rounding rules are on the floating point page.
Round a cloud of gradients to 16 bits
Typical gradient, 10ᵏ, k
-8
Loss scale
Format
Reset
2000 gradients, log-normal around 10^-8, multiplied by loss scale 1 and rounded to fp16: 1584 flush to zero (79.2%), 0 overflow to infinity, 416 survive with mean relative rounding error 3.28e-1. fp16: smallest subnormal 5.96e-8, smallest normal 6.10e-5, largest finite 6.550e+4.

The bars count what happens to 2,000 gradients after they're multiplied by the loss scale and rounded to 16 bits: flushed to zero, kept, or overflowed to infinity. Move the typical gradient size, the loss scale and the format. The gradients are seeded random values spread log-normally around 10ᵏ with a standard deviation of 0.6 decades, and the rounding follows the format's real rules, including its subnormal range. At the default, gradients around 10⁻⁸ in float16 with no scaling, 1,584 of 2,000 vanish; a scale of 2¹⁶ lifts them all into range. Push the typical gradient to 10⁰ with the same scale and about half overflow, which is the failure a dynamic scaler backs off from. bfloat16 keeps every one of them at any setting on this slider, at a mean relative rounding error about eight times float16's.
