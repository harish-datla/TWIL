# Keep a reference, and keep softmax finite

Inference means running a trained model on new inputs: a chatbot reading your prompt and picking its next word, say. Making that cheaper (smaller number formats, faster kernels, a different sampling rule) changes the answers a little, and each change touches a different part of the system. So before you optimize anything, save the unmodified model's outputs on some real inputs. Every change on this page gets measured against that reference.

The first thing that can go wrong isn't an optimization at all. A language model's last layer produces one raw score per token in its vocabulary, called a logit. Softmax turns logits into probabilities: raise e to each score, then divide each result by the total so they add up to 1. Raise e to a large score, though, and the result is too big for a floating-point number. It becomes infinity, and infinity divided by infinity is NaN.

Worked example

Logits (2, 1, 0). e² = 7.39, e¹ = 2.72, e⁰ = 1, total 11.11. Probabilities: 0.665, 0.245, 0.090.

Add 1000 to every logit: (1002, 1001, 1000). The largest 64-bit float is about 1.8 × 10³⁰⁸, which is e709.8, so e1002 overflows and every probability becomes NaN.

Subtract the largest logit first: (0, −1, −2). e⁰ = 1, e⁻¹ = 0.368, e⁻² = 0.135, total 1.503. Probabilities: 0.665, 0.245, 0.090. Same answer, no overflow.

That shift is the standard fix. It works because adding the same number to every logit multiplies the top and bottom of each fraction by the same factor, which cancels.[1]
softmax_i(z) = exp(z_i − max(z)) / Σ_j exp(z_j − max(z))

In words: subtract the biggest logit from every logit, raise e to each result, and divide each by their total. The biggest shifted logit is 0, so no exponential is larger than 1 and nothing overflows.

The formula needs at least one finite logit. If every token is masked out with a score of −∞, there's nothing to normalize, so check for that case explicitly. Very negative shifted logits can underflow to exactly 0; that's a different effect from NaN, and it just means those tokens get probability 0.
Add a constant without changing the intended distribution
Common logit offset
0
Reset
Common offset 0; naive finite; stable probabilities [0.64391, 0.23688, 0.08714, 0.03206]; stable sum 1.00000000. Browser Number uses binary64.

Both panels show softmax of the logits [2, 1, 0, −1] plus the offset. Drag the offset up. The left panel exponentiates directly; from an offset of 710 it overflows in the browser's binary64 arithmetic and turns into an error. The right panel subtracts the maximum first and never changes. The fp16 format overflows much sooner: its largest value, 65,504, is only about e¹¹.
What the shift can't fix

When you port a model or change its number format, compare the intermediate arrays as well as the final answer: each layer's output and the final probabilities. Two versions can pick the same top token while their probabilities differ a lot. And when two tokens are nearly tied, a tiny rounding difference can flip which one has the highest score, the argmax, without anything being badly wrong. Check both the numbers and the task result.
