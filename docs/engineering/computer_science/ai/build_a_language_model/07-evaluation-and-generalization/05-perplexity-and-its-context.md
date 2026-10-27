# Perplexity and its context

A language model's job is to predict the next token, so the natural test is to hand it a text it hasn't seen and ask how much probability it gave each actual next token. High probability on what really came next means a good model. Perplexity turns that into one number.[9]

The score per token is its negative log-likelihood: −log of the probability the model gave the token that actually came next, given every token before it. Averaged over a text, that's the cross-entropy loss from the probability page. Perplexity is e raised to that average, which turns it back into a count of choices.

Worked example

A model gives three successive tokens probabilities 0.5, 0.25 and 0.125.

Negative log-likelihoods: 0.693, 1.386, 2.079. Average: 1.386 nats. Perplexity = e1.386 = 4. In base 2 the average is exactly 2 bits per token.

Perplexity 4 means the model was, on average, as unsure as if it were picking uniformly among 4 tokens.

Written out, for a text of t tokens:
PPL(X) = exp(−(1/t) Σi log p(xi | x<i))

In words: average the negative log-probability the model gave each actual token, then exponentiate. Lower is better, and 1 would mean the model was certain and right every time.

Two things change the number without changing the model. The first is the tokenizer. Perplexity is per token, so a tokenizer that cuts the same text into more pieces divides the same total by a bigger count. Bits per character, the total negative log-likelihood in bits divided by the number of characters, doesn't depend on the tokenizer.[9]

Worked example

A 20-character text gets a total negative log-likelihood of 10 nats under two models that assign it the same overall probability. One tokenizer splits it into 5 tokens: 2 nats per token, perplexity e² ≈ 7.4. The other uses 10 tokens: 1 nat per token, perplexity e ≈ 2.7.

Bits per character: 10 nats ÷ ln 2 ≈ 14.4 bits, ÷ 20 characters ≈ 0.72 for both.

The second is how much context each token gets. A model with a fixed window, evaluated on back-to-back chunks that don't overlap, scores the first tokens of each chunk with little or no context, which raises the perplexity. A sliding window that moves forward a few tokens at a time (its stride) gives every token a long context, at the cost of more forward passes.[9]
Score a sentence under a trigram model with and without context
Test sentence
Window
8
Windows
Reset
Character trigram model trained on a 200-character text; evaluated on "the brown dog runs over the quiet field while the quick fox sleeps" (66 characters). Window 8, non-overlapping: mean NLL 2.4341 nats, perplexity 11.406, 3.512 bits per character. Unlimited context: perplexity 10.283. The gap is the characters scored at window starts with less context than the model could use.

The bars show the negative log-likelihood of each character in the test sentence; taller means the model was more surprised. Change the window size and switch between non-overlapping and sliding windows. The model is a character trigram model with add-one smoothing, trained on a 200-character text, and it scores each character given up to two previous ones. With unlimited context the first sentence has perplexity 10.28, 3.36 bits per character. Non-overlapping windows of 8 restart the context every 8 characters: the spikes in the bars are those positions, and the perplexity rises to 11.41. Windows of 2 raise it to 15.1. The sliding variant with stride 4 gives every character its full two-character context and matches the unlimited figure, with the same model throughout.
