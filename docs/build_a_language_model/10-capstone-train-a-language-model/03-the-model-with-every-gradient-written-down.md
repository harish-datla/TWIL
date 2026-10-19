# The model, with every gradient written down

With batches ready, the model needs two passes. The forward pass turns 32 characters into 32 next-character predictions. The backward pass works out, for every one of the model's 15,264 numbers, which way to nudge it to lower the loss. This page writes both out by hand, with no autodiff library, so every step is visible.

The architecture is nanoGPT's, cut down to one transformer block and no bias vectors.[7] For each position it does four things:

    Look up the token's row in the token embedding table and the position's row in the position embedding table, and add them. The sum, a vector of 32 numbers, is the residual stream: the running vector that each later step reads from and adds to.
    Normalize the stream with LayerNorm, which shifts and scales its 32 numbers to mean 0 and spread 1. Run causal attention on the result, which lets the position take a weighted mix of information from itself and earlier positions but never later ones. Add attention's output back to the stream.
    Normalize again and run a small two-layer network, the MLP: widen the vector to 128 numbers, apply GELU (a smooth cousin of ReLU that passes positive inputs and squashes negative ones toward zero), and narrow it back to 32. Add that to the stream too.
    Normalize once more and score every character. The score for character v is the dot product of the final vector with v's own row of the token embedding table, so the input table doubles as the output layer. That reuse is weight tying.

In symbols, the forward pass is:
x = Etok[id] + Epos[t]
x ← x + Wproj·Attn(LN1(x)) x ← x + W2·GELU(W1·LN2(x))
z = Etok·LNf(x) p = softmax(z)

In words: start from the token's embedding plus its position's embedding, add attention's output, add the MLP's output, normalize, then score each character by its dot product with that character's embedding and turn the scores into probabilities with a softmax.

Attn is causal multi-head attention as built on the Transformers page. One 32 × 96 matrix (C × 3C) turns each position's vector into a query, a key and a value. A query-key dot product scores how relevant each earlier position is, the scores are divided by √(head width) and softmaxed into weights, and the output is the weighted sum of the values. Here 4 heads of width 8 do this side by side and their outputs are joined back into 32 numbers. L is the number of blocks: 1 here, 12 for GPT-2 124M.

Weights start as small random numbers drawn from a normal distribution with mean 0 and standard deviation 0.02, written N(0, 0.02). The two matrices that write into the residual stream, Wproj and W2, start smaller, at N(0, 0.02/√(2L)): the stream receives 2L such additions, and shrinking each one keeps the stream's variance in check. The LayerNorm gains start at 1.[7]

Worked example

58 characters, context 32, width 32, 4 heads of width 8:

Token embedding 58 × 32 = 1,856. Position embedding 32 × 32 = 1,024. Attention 32 × 96 + 32 × 32 = 4,096. MLP 32 × 128 + 128 × 32 = 8,192. Three LayerNorm gains of 32 = 96.

Total: 15,264 parameters.

The trainer has no autodiff. Each layer's backward pass is written out, which is the job a tape does for you on the Automatic Differentiation page. Writing them out also makes them testable with a finite-difference check: nudge one weight by a tiny amount, rerun the forward pass, and the change in the loss divided by the nudge should match the gradient. The listings below run exactly that check. Most of the backward pass is matrix multiplication run in reverse. Three pieces need real derivations.

The loss at the output. Start at the top: how should each of the 58 scores at a position change to lower the loss? For a softmax followed by cross-entropy the answer is short: the predicted probability, minus 1 for the right character.

Worked example

Three characters a, b, c with logits (2, 1, 0). Softmax: p = (0.665, 0.245, 0.090). The right answer is b, so the loss is −ln 0.245 = 1.41.

Gradient = p minus one-hot(b) = (0.665, 0.245 − 1, 0.090) = (0.665, −0.755, 0.090).

Gradient descent moves against it: b's score goes up, a's goes down the most because it took the most probability by mistake. The entries sum to zero.
∂L/∂zv = (pv − 𝟙[v = y]) / (B·T)

In words: each score's gradient is its probability minus 1 if it's the right answer (minus 0 otherwise), divided by the number of positions B·T because the loss is their mean. A confident wrong answer pushes hard. A confident right answer, with p near 1, barely pushes at all. Checking this sign and size is the first step when gradients look wrong.

Attention's softmax. Attention also turns a row of scores into weights with a softmax, but there's no one-hot target there. Instead, a gradient gj arrives from above for each weight: how much the loss would change if weight j grew. The score gradient takes each weight's g, subtracts the weighted average of all the g's, and scales by the weight.

Worked example

Weights w = (0.5, 0.3, 0.2) and incoming gradients g = (1, 0, 0). Weighted average of g: 0.5 × 1 = 0.5.

Score gradients: 0.5 × (1 − 0.5), 0.3 × (0 − 0.5), 0.2 × (0 − 0.5) = (0.25, −0.15, −0.10).

They sum to zero. Adding the same amount to every score leaves a softmax unchanged, so the gradient never points in that direction.
∂L/∂sj = wj ( gj − Σk wk gk )

In words: each score's gradient is its weight times its own incoming gradient minus the weighted average of all of them. Masked positions have weight 0, so they get no gradient. From there the score's gradient splits into the query and the key through the scaled dot product that made the score, and each value's gradient is its weight times the gradient arriving at the attention output.

LayerNorm. LayerNorm needs the most care, because every output depends on every input through the mean and the spread. The forward pass throws away two things: the vector's average and its overall scale. Add 5 to every input, or double every input, and the output doesn't change (apart from the tiny ε added for safety). So any part of the incoming gradient that asks for one of those two changes can't do anything, and the backward pass removes it.

Worked example

x = (1, 2, 3): mean 2, σ = 0.816, so the normalized x̂ = (−1.22, 0, 1.22), ignoring ε. Take gain γ = 1, so ĝ is just the incoming gradient.

ĝ = (1, 1, 1) asks to raise every output, which LayerNorm can't do: gradient (0, 0, 0). ĝ = x̂ asks to spread the outputs further, which it also can't do: gradient (0, 0, 0).

ĝ = (1, 0, 0) gives (0.204, −0.408, 0.204), which matches a finite-difference check to six decimals.
∂L/∂x = (1/σ) ( ĝ − mean(ĝ) − x̂ · mean(ĝ ⊙ x̂) ) ∂L/∂γ = Σpositions ∂L/∂y ⊙ x̂

In words: take the incoming gradient scaled by the gain, subtract its mean (the "shift everything" part) and its projection onto x̂ (the "scale everything" part), and divide by the input's spread σ. The gain's own gradient is the incoming gradient times x̂, summed over positions.

The variance is the biased estimate (divide by C, not C − 1) and ε sits inside the square root, as in PyTorch.[8] GELU's derivative comes straight from its formula; the page uses the tanh approximation that PyTorch documents alongside the exact x·Φ(x).[9]
