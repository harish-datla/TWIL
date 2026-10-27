# Information and entropy

Training logs and papers report a language model's loss in bits or nats per token. To read that number you need a way to measure surprise. If a model gave the next token 99% and that token appears, the model is barely surprised. If it gave 1% and that token appears, it's very surprised. Information theory turns that surprise into a number.

The surprise of an outcome with probability p is its self-information, −log₂ p, measured in bits. One bit is the surprise of a fair coin flip. The logarithm is there so that surprises add up: the probabilities of independent events multiply, and the log turns that product into a sum.[2]

Worked example

A fair coin, p = 1/2: −log₂ 0.5 = 1 bit. Two fair coins both landing heads, p = 1/4: 2 bits, one per coin. A 1-in-1,024 event: 10 bits.

A token the model gave 1%: −log₂ 0.01 ≈ 6.6 bits. A token it gave 97%: about 0.04 bits.

A distribution's entropy is its average surprise: weight each outcome's self-information by the outcome's probability and add.[6] It measures how unpredictable a draw from the distribution is.

Worked example

A fair coin: 0.5 × 1 + 0.5 × 1 = 1 bit.

A coin that lands heads 90% of the time: heads carries −log₂ 0.9 ≈ 0.152 bits and tails −log₂ 0.1 ≈ 3.32 bits, so the entropy is 0.9 × 0.152 + 0.1 × 3.32 ≈ 0.47 bits. It's easier to predict, so each flip tells you less on average.

The formula is that weighted average, written for any number of outcomes.
H(p) = −Σx p(x) log2 p(x) = Ep[−log2 p(x)]

In words: entropy is the average number of bits of surprise per draw. It's 0 when one outcome is certain and largest, log₂ k bits, when all k outcomes are equally likely.[1][2]

With the natural logarithm in place of log₂ the unit is the nat, about 1.44 bits. Frameworks such as PyTorch compute losses with natural logs, so the loss they report is in nats.
Reshape a four-outcome distribution
P(A), %
25
P(B), %
25
P(C), %
25
Reset
p = (0.250, 0.250, 0.250, 0.250). Self-information −log₂p = (2.00, 2.00, 2.00, 2.00) bits. Entropy H = Σ p·(−log₂p) = 2.0000 bits; maximum log₂4 = 2 bits at the uniform distribution. Zero-probability outcomes contribute 0 to H.

Drag A, B and C to reshape a distribution over four outcomes. D takes whatever is left, and if the three sliders pass 100% all four are rescaled to sum to one. The left bars are the probabilities and the right bars are each outcome's surprise in bits. Uniform gives 2 bits, the maximum for four outcomes. Push A toward 100% and B and C toward 0, and the entropy falls toward zero (97%, 1%, 1%, 1% gives 0.24 bits) even though each rare outcome's surprise keeps growing. Each rare outcome is very surprising, but it carries almost no weight in the average.

Text has entropy too. What a person types next is partly unpredictable, and no model can remove that randomness. So the entropy of the text, in bits per token, is a floor under any model's loss on it. Training can only remove the extra surprise a model adds on top. The scaling-law fits on the training page include a constant term for this irreducible loss.
Where the word "bit" comes from

