# AI Inference and Quantization from Scratch

[Back to Build a Language Model](../index.md)

This tutorial covers stable softmax, integer quantization, zero points, quantized dot products, roofline reasoning, sampling controls, and measuring inference performance.

## Sections

- [Keep a Reference, and Keep Softmax Finite](01-keep-a-reference-and-keep-softmax-finite.md)
- [Store Each Number as a Small Integer](02-store-each-number-as-a-small-integer.md)
- [One Big Value Can Spoil Everyone’s Range](03-one-big-value-can-spoil-everyones-range.md)
- [Subtract the Zero Points Before You Multiply](04-subtract-the-zero-points-before-you-multiply.md)
- [Is the Layer Waiting on Memory or Math?](05-is-the-layer-waiting-on-memory-or-math.md)
- [Temperature and Top-p Reshape the Next-Token Odds](06-temperature-and-top-p-reshape-the-next-token-odds.md)
- [A Checked Integer Dot Product](07-a-checked-integer-dot-product.md)
- [Measure What Actually Ran](08-measure-what-actually-ran.md)
- [What’s Next](09-whats-next.md)
