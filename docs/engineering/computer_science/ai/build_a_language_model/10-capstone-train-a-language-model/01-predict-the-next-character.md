# Predict the next character

Your phone's keyboard suggests the next word while you type. A language model does the same job, one step at a time. On this page the unit of text, the token, is a single character, so the model's whole job is this: look at the characters so far and give each of the 58 possible characters a probability of coming next. To write a paragraph, it picks a character, appends it and asks again.

That's what a language model is: a model that assigns a probability to any piece of text, built up from one next-token prediction per position.[1] Training needs one number that says how good those predictions are. The usual choice is the cross-entropy loss. At each position, find the probability the model gave to the character that actually came next and take minus its natural log. A confident right answer costs almost nothing: probability 0.9 costs 0.11. A confident wrong answer, where the right character got probability 0.01, costs 4.6. The loss is the average cost over every position in the batch.

Worked example

A window of three positions. The model gives the character that really came next a probability of 0.5, then 0.25, then 0.1.

Costs: −ln 0.5 = 0.693, −ln 0.25 = 1.386, −ln 0.1 = 2.303. Loss = their mean = 1.461 nats (a nat is the unit you get from natural logs).

exp(1.461) = 4.31: on average the model was as unsure as if it were picking among about 4.3 equally likely characters.

The equation below is that calculation for a window of T characters, plus the conversion from loss to "number of equally likely choices", which is called perplexity.
L = −(1/T) Σt log pθ(xt+1 | x1 … xt) PPL = exp(L)

In words: at each position t, take the probability the model gave to the character that really came next after seeing the first t characters, take its log, average over the window and flip the sign. Perplexity is e raised to that loss. Divide the loss by ln 2 to get bits per character instead of nats.

A loss means little until you compare it with something. Three yardsticks do that. A model that knows nothing gives all 58 characters probability 1/58, so it scores ln V = ln 58 = 4.060 at every position. That's also where a correctly built model should start. Its weights begin small, so its raw scores for each character (the logits, which a softmax turns into probabilities) start near zero, and near-zero logits give near-uniform probabilities.[1][2]

The other two yardsticks are simple counting models. One ignores context and predicts each character by how often it appears in the training text: it scores 3.215 on this page's validation text. The other, a bigram model, looks only at the previous character and predicts from counts of character pairs: it scores 2.615. Both add 0.5 to every count, so a character or pair never seen in training still gets a little probability. A model that beats 2.615 is using more than one character of context. The page's perplexity readouts report exp of the loss: 4.060 nats is a perplexity of 58, and 2.4 nats is about 11.
