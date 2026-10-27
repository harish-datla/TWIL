# Training on pairs that should match

Co-occurrence gets you words that fill the same slots in sentences. A search system needs a different relation: the question "how do I reset my router?" should land near the passage "hold the button on the back for ten seconds", though the two share no words. Modern sentence and image embedding models learn that relation from pairs that should match, such as a question and its answer, a caption and its image, or a sentence and its translation.

The training trick is to use the rest of the batch as wrong answers. Take N pairs. Embed all N questions (the anchors) and all N answers (the positives), and compute the cosine similarity of every anchor with every positive, which gives an N×N grid. Row i should have its biggest value in column i, its own answer. The in-batch contrastive loss treats each row as a multiple-choice question with N options and scores it with a softmax, so every other answer in the batch is a free wrong option, called a negative.[6]

Before the softmax, the similarities are divided by a temperature τ, a small positive number. A small τ stretches the gaps between scores, so the loss concentrates on the wrong answers that score almost as high as the right one. CLIP, OpenAI's image–text model, multiplies its cosines by a learned scale that starts at 1/0.07 ≈ 14.3, which is the same as a temperature of 0.07.[7]

Worked example

N = 3. Row 1's cosines with the three answers are 0.5 (its own), 0.4 and 0.1.

At τ = 1, the softmax of (0.5, 0.4, 0.1) gives the right answer a probability of 0.39, so the loss is −log 0.39 = 0.95.

At τ = 0.1, dividing first gives (5, 4, 1). The right answer now gets 0.72 and the loss drops to 0.33. Of the remaining 0.28, almost all sits on the 0.4 answer, the near miss, so that's the negative the update pushes away hardest.

Written out, the loss for row i is the cross-entropy of that softmax. The batch loss is its average over all N rows.
ℓi = −log exp(sii / τ) / Σj exp(sij / τ)

In words: turn row i into probabilities and take −log of the probability on the anchor's own positive. The loss is near 0 when that positive takes almost all the probability, and grows when other answers in the batch compete with it.

If the similarities carry no information, every row is uniform: each of the N candidates gets probability 1/N and the loss is log N. For N = 8 that's log 8 ≈ 2.08 nats. That chance level is the grey line in the widget.
Pull pairs together, push the rest apart
Pairs in batch N
Temperature τ
0.1
Step
Play
Reset
8 pairs, temperature 0.1, 0 gradient steps. Loss 1.2586 nats; a model that cannot tell pairs apart scores log 8 = 2.079. Mean cosine of matched pairs 0.764, of mismatched pairs -0.036.

Click Step (five gradient updates) or Play and look at the left grid. Rows are anchors, columns are positives, brighter means a higher cosine, and the outlined diagonal is each anchor's own match. Each anchor starts as a random 4-dimensional vector and its positive as a noisy copy; the updates move both sides through the normalized cosine. The loss starts near 1.26 nats for N = 8 and falls below 0.05 as the diagonal brightens and the off-diagonal entries dim. The grey line on the right is log N, the loss of a model whose similarities carry no information. Larger batches raise that line and supply more negatives per step; Sentence Transformers offers a cached variant of the loss to raise the batch size past what memory would otherwise allow, reporting better results from the larger batch.[6]
Check Your Understanding Q1 of 2

Why does word2vec draw noise words from counts raised to 0.75 rather than from the raw counts?
To make the sampling table smaller.
To flatten the distribution so rare words are drawn as negatives more often than their frequency gives.
Because 0.75 is the fraction of pairs that are negatives.
0/2 correct
