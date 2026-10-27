# Guess cheaply, check in one pass

A memory-bound decode step leaves most of the GPU's arithmetic idle, and checking several tokens in one pass costs little more than producing one. Speculative decoding uses that spare arithmetic. A cheap draft guesses the next γ tokens. The big target model then scores all of them in one forward pass and keeps the longest run of guesses it agrees with. It also adds one token of its own: the correct one at the first disagreement, or the next one after a full match.[14][1]

The output doesn't change. With greedy decoding it's exactly what the target alone would produce; with sampling, a rejection rule keeps the output distribution the same.[14] The draft can be a smaller model with the same tokenizer, extra prediction heads on the target, or n-gram lookup in the prompt itself, which TGI offers for code and repetitive text.[15][16]

Worked example

The draft's guesses are accepted 80% of the time, and it guesses γ = 4 tokens per round.

Expected tokens per target pass: (1 − 0.8⁵) / (1 − 0.8) = (1 − 0.328) / 0.2 = 3.36.

If each draft token costs a tenth of a target pass, a round costs 4 × 0.1 + 1 = 1.4 target passes, so the speedup is 3.36 / 1.4 = 2.4×.

Leviathan et al. derive the expected yield assuming every guess is accepted independently with the same probability α; dividing by the round's cost gives the speedup.[14]
E[tokens] = (1 − αγ+1) / (1 − α) speedup = E[tokens] / (γ·c + 1)

In words: each round yields the accepted guesses plus the target's own token. A guess counts only if every guess before it was accepted, so the yield is a sum of shrinking powers of α. The speedup is that yield divided by what the round costs: γ draft tokens plus one target pass.

The formula assumes a target pass over γ + 1 tokens costs about the same as a pass over one. That holds while decode is memory-bound, at small batches. At large batches the extra tokens cost real arithmetic, and TensorRT-LLM's documentation says speedups are visible only at low batch sizes.[15]
Measure acceptance with an n-gram draft against an n-gram target
Draft
Draft cost c
0.1
Reset
Target: character 6-gram; draft: trigram; 400 characters generated greedily from "The ". Every run reproduces the target's own output exactly (lossless: true). Per-token acceptance at γ = 1: 0.68. Tokens per target call: γ=1 → 1.67, γ=2 → 2.42, γ=3 → 2.42, γ=4 → 3.15, γ=6 → 3.48, γ=8 → 3.88; the independent-acceptance formula with α = 0.68 predicts 1.68, 2.14, 2.45, 2.66, 2.90, 3.01. With a draft costing 0.1 of a target pass, the best measured speedup is 2.25× at γ = 4.

The left panel shows measured tokens per target call (purple) against the formula's prediction (green) for draft lengths γ from 1 to 8; the right turns the measurements into a speedup at draft cost c. Switch drafts and drag the cost. The target is a character-level 6-gram model and the draft a shorter n-gram model, both fit to the farm and space text of the fine-tuning page. Each run generates 400 characters greedily and reproduces the target's own output character for character. With the trigram draft, acceptance at γ = 1 is 0.68 per token, and the measured yield is 1.67 tokens per call at γ = 1 and 3.88 at γ = 8. The formula with that α predicts only 3.01 at γ = 8, because acceptances aren't independent: a draft that has been right for several characters is inside a phrase the target also knows, and a draft that's wrong is often wrong for a reason the next character shares. The 4-gram draft accepts 88% and reaches 7.8 tokens per call at γ = 8. The bigram draft stalls at about 1.7 whatever γ is, and with a draft costing a tenth of a target pass its speedup drops below 1 at γ = 8. Transformers avoids choosing γ by hand: it lengthens the draft after a full match and shortens it after a miss.[1]
Check Your Understanding Q1 of 2

Speculative decoding with a draft that is accepted 80% of the time and γ = 4 yields how many tokens per target pass on average, and at what batch size does the gain disappear?
About 3.4 tokens; the gain disappears once the target's decode step is compute-bound.
Exactly 4 tokens; the gain is independent of batch size.
About 0.8 × 4 = 3.2 tokens; the gain disappears only when the draft is slower than the target.
0/2 correct
