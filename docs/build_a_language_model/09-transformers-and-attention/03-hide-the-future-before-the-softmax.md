# Hide the future before the softmax

A language model trains on ordinary text by predicting each next token from the ones before it, and it makes all those predictions in one pass. If position 2 could attend to position 3, it could read the answer instead of predicting it. Causal attention prevents that: each position may attend to itself and to earlier positions, never later ones.[2]

The exclusion has to happen in the scores, before the softmax. Set each forbidden score to −∞; its exponential is 0, so it gets no weight and the allowed weights still sum to 1. Two tempting shortcuts are wrong. Setting a forbidden score to 0 still gives it weight, because exp(0) = 1. Zeroing weights after the softmax leaves the remaining weights summing to less than 1.

Worked example

A query's scores against tokens A, B and the future token C are 1.0, 2.0 and 3.0.

Mask C with −∞: the softmax over (1.0, 2.0) gives A 0.27 and B 0.73, and C gets 0.

Replace C's score with 0 instead: the softmax over (1.0, 2.0, 0) gives 0.245, 0.665 and 0.09, so 9% of the output still comes from the future token.
Edit a future value and inspect leakage
Query row
2
Last-token value
4
Mask
Reset
Query row 2; causal. Last-token weight 0.000000. Output causal [0.078, 0.687]; unmasked [1.001, -0.458].

Pick a query row, drag the last token's value, and switch between Causal and Unmasked. Yellow marks the selected row of the weight matrix; entries are weights after the softmax, not raw scores. With the causal mask, changing the last value leaves rows 0 to 3 untouched, because their weight on the last token is exactly 0. Row 4 is the last token itself, which may attend to itself, so its output does change. Unmasked, the edit changes every row's output.

A position may attend to itself because the targets are shifted by one. For the input A, B, C the targets are B, C, D: the position holding B is asked to predict C, so reading A and B is fair. Padding adds a second rule. When short sequences are padded to a common length, a separate mask marks the padding positions, and it's combined with the causal rule so padding gets no weight either.

In training, every position is computed in parallel under the mask. Generation is different: the next token isn't known until the model has produced the previous one, so autoregressive decoding still runs one token at a time.[2] Softmax over a row with every score masked is undefined, so an implementation that can produce such a row needs an explicit rule for it.
