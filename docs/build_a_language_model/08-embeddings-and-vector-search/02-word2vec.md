# word2vec: learn vectors from the company words keep

Nobody hands you a table saying cat is close to dog. What you do have is a lot of text, and in text, "cat" and "dog" turn up in the same kinds of sentences: "the ___ sleeps in the barn". word2vec turns that into a training task. It learns vectors that predict which words appear near which, and words used in the same contexts end up with similar vectors.[1]

The variant on this page is called skip-gram. Slide a window along the text. The word in the middle is the center word, and each word within a couple of positions of it is a context word. Every word gets two vectors, one used when it's the center and one used when it's context. The model scores a (center, context) pair by the dot product of those two vectors, and training pushes the scores of pairs that really occur upward.

Worked example

Sentence: "the cat sleeps in the barn". With a window of 2, the center word "sleeps" produces four pairs: (sleeps, the), (sleeps, cat), (sleeps, in) and (sleeps, the). Every other position in the sentence produces its own pairs the same way.

The original skip-gram turns those scores into probabilities with a softmax over the whole vocabulary, a function that makes every word's score positive and scales them so they sum to 1.[1] That's the expensive part. With a vocabulary of 100,000 words, every training pair needs 100,000 dot products, and its update touches all 100,000 context vectors.

Negative sampling replaces that one big choice with a few yes-or-no questions. For each real pair, also draw K random "noise" words, 5 by default in word2vec. The model should answer yes for the real pair and no for each noise word. Each answer comes from the sigmoid σ, which squashes any score into a probability between 0 and 1. The cost per pair now grows with K instead of with the vocabulary size.[3]

Worked example

Center word "cat", real context word "eats", K = 2 noise words "garage" and "the". Say the current dot products are 2.0, 0.5 and −1.0.

Real pair: σ(2.0) = 0.88, so its loss is −log 0.88 = 0.13.

Noise word "garage": the right answer is no, so it's scored on σ(−0.5) = 0.38, a loss of −log 0.38 = 0.97. Noise word "the": σ(1.0) = 0.73, a loss of 0.31.

Total loss ≈ 1.41. The biggest piece comes from "garage", whose score is too high, and that's where the next update pushes hardest.

The equation below is that loss for one real pair and K noise words.
ℓ = −log σ(uoᵀvc) − Σk=1..K log σ(−ukᵀvc)

In words: score the real pair and push its probability toward 1; score each noise word and push its probability toward 0. The loss adds up −log of those probabilities, so it's small only when the real pair scores high and every noise word scores low.
Train skip-gram vectors on a two-topic corpus
Seed
2
Step
Play
Reset
2400 tokens, 28-word vocabulary, 8-dimensional vectors, window 2, 5 negatives, seed 2. Epochs trained: 0. Nearest by cosine: cat → on (0.67), field (0.60), parks (0.33); car → runs (0.64), cow (0.34), parks (0.33); barn → truck (0.41), drives (0.34), the (0.32).

Click Step to train for one epoch, or Play to keep going, and look at the left panel. Each dot is a word's center vector, flattened to the two directions where the vectors vary most: farm words green, traffic words purple, shared words grey. The corpus is 400 templated six-word sentences, half about farm animals and half about traffic, sharing "the", "in", "on" and "near". One epoch is a pass of negative-sampling updates over every (center, context) pair in a window of 2. Before training the dots are a random cloud. After a few epochs the farm and traffic words separate, and "cat" lists only farm animals ("cow", "dog", "horse" at seed 2) as its nearest neighbours. "barn" doesn't: from the second epoch on, at least two of its three nearest neighbours are traffic places such as "garage", "road" and "station", because the corpus puts every place word in the same slot, after a preposition and "the". Skip-gram similarity measures which words fill the same slots, and a barn and a garage fill the same slot while naming different things.

Which words get drawn as noise matters. Drawing them in proportion to how often they appear would mostly pick "the" and "in", so rare words would almost never be pushed away from anything. word2vec instead draws each word in proportion to its count raised to the power 0.75. That flattens the distribution, so rare words are drawn more often than their frequency alone would give.[4][5]

Worked example

In the widget's corpus "the" appears 800 times and "bike" 32 times, a ratio of 25. Raised to 0.75 they become 8000.75 ≈ 150 and 320.75 ≈ 13.5, a ratio of about 11. "the" is still the most likely noise word, but its lead over "bike" has shrunk by more than half.
Flatten the noise distribution with the 0.75 exponent
Exponent
0.75
Reset
Counts in the corpus: "the" 800, "bike" 32. Noise distribution ∝ count^0.75: P(the) = 0.210, P(bike) = 0.0188; ratio 11.2 versus the raw count ratio 25.0. Exponent 1 is the unigram distribution; exponent 0 is uniform.

Drag the exponent and compare the bars, which show the probability that each word is drawn as a noise word. Counts come from the widget's corpus: "the" appears 800 times, the rarest word 32 times, a ratio of 25. At exponent 1 the noise distribution is the raw word frequencies and "the" is drawn a third of the time. At 0.75 its share drops to 21% and the ratio to the rarest word to about 11, so rare words get pushed away from unrelated centers more often. At 0 every word is drawn equally often, so "the" is drawn as a negative no more often than the rarest word.
word2vec's real default settings

