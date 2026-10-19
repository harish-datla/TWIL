# Temperature and top-p reshape the next-token odds

A chatbot's model gives a probability for every possible next token, and something still has to pick one. Always taking the most likely token, called greedy decoding, tends to produce repetitive text; sampling from the full distribution sometimes picks nonsense from the long tail of unlikely tokens.[8] Two settings sit between those extremes.

Temperature T divides every logit before softmax. T below 1 widens the gaps between logits, so the top token gets more of the probability. T above 1 shrinks them and flattens the distribution. T = 0 would divide by zero, so it's handled as a separate rule: take the top token.

Top-p, or nucleus sampling, sorts tokens by probability and keeps the smallest set from the top whose probabilities add up to at least p. It rescales those to sum to 1 and samples from them. The token that pushes the total past p stays in. Unlike top-k, which always keeps k tokens, the number kept follows the distribution: a confident model keeps few, an unsure one keeps many.[8]

Worked example

Logits (3, 2, 1, 0, −1, −2). At T = 1, softmax gives 0.634, 0.233, 0.086, 0.032, 0.012, 0.004.

At T = 0.5 the logits double to (6, 4, 2, 0, −2, −4), and the top token's probability rises to 0.865.

Top-p with p = 0.9 at T = 1: running totals 0.634, then 0.867 (still under 0.9), then 0.953, which crosses it. Keep three tokens. Divide each by 0.953: 0.665, 0.245, 0.090.
Inspect temperature and nucleus filtering
Temperature T
1
Nucleus mass p
0.9
Reset
T=1, top-p=0.9; retained 3/6 tokens, pre-renormalization mass 0.95257; output sum 1.000000. No draw is needed to inspect the sampling distribution.

The left panel shows the probabilities of six tokens with logits [3, 2, 1, 0, −1, −2] after temperature; the right shows what's left after the top-p cut, rescaled to sum to 1. At the defaults (T = 1, p = 0.9) three tokens survive, holding 0.953 of the probability before rescaling. Lower T or p and fewer survive; raise them and more do. Temperature is applied first, then the cut. The panels show the exact probabilities, so no random draws are needed.

Neither setting fixes a wrong model. If the logits favor a wrong answer, a lower temperature makes the model pick it more often. Cutting the tail changes how varied the output is and how often a long-shot token appears; it doesn't make the text more factual. Holtzman et al. studied how decoding rules change generated text, and their results don't name one best setting for every application.

Write down the order of the steps. Cutting the tail before applying temperature can keep a different set of tokens than applying temperature first. Break ties in a fixed way if you need reproducible output, and check that at least one token survives. Keep the model's probabilities and the sampler's final probabilities as separate arrays.
