# Scaling laws and the compute budget

Say you can afford 10²¹ floating-point operations of GPU time. You could train a big model on a modest amount of text, or a smaller model on much more. Both cost the same; which one ends up with the lower loss? Scaling laws answer that from measured training runs instead of guesswork.

Start with the cost. Training a transformer with N parameters on D tokens takes about 6ND floating-point operations (FLOPs): about 2 per parameter per token for the forward pass, and twice that for the backward pass.[15] A compute budget C therefore fixes the product ND, and the only choice left is how to split it.

Worked example

A 1-billion-parameter model trained on 20 billion tokens costs 6 × 10⁹ × 2×10¹⁰ = 1.2×10²⁰ FLOPs.

With 10²¹ FLOPs you could train that model on about 167 billion tokens, or a 10-billion-parameter model on about 17 billion.

Scaling laws pick the split by measurement. Hoffmann et al., in the paper known as Chinchilla, trained hundreds of models, fit their final loss as a function of N and D, and found that at a fixed budget model size and token count should grow together. That implied the large models of the time had been trained on too few tokens for their size.[14] Their fitted formula predicts the final loss from the model size and the token count, which is exactly what you need to compare two ways of spending one budget:
L(N, D) = E + A/Nα + B/Dβ

In words: the loss has a floor E that no model can get below, plus one penalty that shrinks as the model grows and another that shrinks as it sees more data. Spending a budget well means balancing the two penalties.

For the 1-billion-parameter model on 20 billion tokens: 406.4/(10⁹)0.34 ≈ 0.354 and 410.7/(2×10¹⁰)0.28 ≈ 0.536, so the predicted loss is 1.69 + 0.354 + 0.536 ≈ 2.58 nats per token.

Treat those constants with care. The paper estimated the best split three different ways; the constants here are its third method's fit, as reproduced in nanoGPT's scaling-law notebook, and the notebook notes that it can't make this fit reproduce the paper's own table of optimal sizes exactly.[15] The widget treats the formula as a fit whose shape is the lesson and draws the table from the paper's second method alongside for comparison.
Spend a compute budget between parameters and tokens
Budget, 10ᵏ FLOPs, k
21
Reset
Budget C = 10^21 FLOPs with C ≈ 6ND. The fit L = 1.69 + 406.4/N^0.34 + 410.7/D^0.28 is minimized at N ≈ 1.84e+9 parameters, D ≈ 9.04e+10 tokens (49 tokens per parameter), loss 2.329. The floor E = 1.69 is the fitted irreducible loss.

Left: the predicted loss for every model size the budget could pay for, L(N, C/6N), with the best one marked. Right: the best token count for each model size, from the fit (line) and from the paper's table (dots). Drag the budget slider and both panels update. Too small a model wastes the budget on tokens it can't use; too large a model sees too few tokens. At 10²¹ FLOPs the fit's minimum is near 1.8 billion parameters on 90 billion tokens. The right panel sweeps budgets from 10¹⁷ to 10²⁶ FLOPs and traces the fit's optimum against the paper's second-method table, which lists 20 billion tokens for a 1-billion-parameter model and 220 billion for 10 billion. Both grow tokens at least in proportion to parameters, but at different rates: the fit's optimum moves from about 26 tokens per parameter at 10¹⁸ FLOPs to about 155 at 10²⁶, while the table stays between 19 and 29. The notebook reports the same kind of mismatch between the fit and the paper's table and leaves it unresolved.
Check Your Understanding Q1 of 2

Why does uncorrected Adam take overly large steps at the start of training?
Because the gradient is largest at initialization.
Because v starts further from its true value than m, so m/√v is biased upward.
Because ε is too small.
0/2 correct
