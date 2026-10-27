# Characters, splits and batches

Before a model can train, the text has to become numbers, and the numbers have to be cut into examples. This page uses the simplest tokenizer that works: every distinct character is a token, and the tokens are numbered in sorted order. nanoGPT's character-level Shakespeare does exactly the same, with 65 symbols over 1.1 million characters there and 58 over 15,957 here.[3]

Worked example

Sorted, the 58 characters start with the newline (id 0), the space (1) and punctuation, then capitals from A (10), then lowercase. "First Cit" becomes 15, 40, 49, 50, 51, 1, 12, 40, 51.

Take a window of T = 8 tokens starting at "F". Inputs: "First Ci". Targets, the same text shifted one place: "irst Cit".

Position 1 sees "F" and should predict "i". Position 8 sees "First Ci" and should predict "t". One window gives 8 predictions to learn from.

The last 10% of the text, in document order, is held out for validation: text the model never trains on, used to check whether it has learned the language or only memorized its training text. Keeping document order matters. Shuffle before splitting and the first half of a sentence can land in training and the second half in validation, so the model gets scored on text it has partly seen. Here that leaves 14,361 training characters and 1,596 validation characters.

A training batch is B windows of T tokens, each starting at a random place in the training text. Because the starting points are random, no window boundary is fixed, and over many batches every stretch of text gets seen. d2l describes the alternative: cut the text into fixed windows from one random offset per pass.[1][4] A batch gives B × T predictions to learn from, 8 × 32 = 256 on this page. nanoGPT prints that number as tokens per iteration, and it's the unit scaling laws count training data in.[4]
Tokenize the excerpt and draw a batch
Window T
Batch seed
1
Reset
15957 characters, 58 distinct → 14361 training and 1596 validation tokens (a 90/10 split in document order). Each batch is 8 windows × 32 tokens = 256 target predictions; one pass over the training split takes about 57 batches. Window offsets for seed 1: 8984, 39, 7557, 14056, 13874, 4027, 8780, 10326. Most frequent: "␣" 2113, "e" 1282, "t" 975, "o" 815, "s" 716.

Left: the twenty most common characters in the training text. Right: three windows from one batch, each shown as its inputs (in) and, below, its targets (out). Move the batch seed to draw other windows, or pick a shorter window T. Space dominates the counts, then e, t and o, with s, a, i, r, h, n and the newline next; the rarest characters appear a handful of times or once, too few for the model to learn confident predictions for them. In the batch panel each target row is the input row shifted one place left: position t of the input predicts position t+1. Showing the model the target as an input, an off-by-one in the slicing, is one of the silent bugs Karpathy's recipe lists.[2] With the target visible, training loss falls far faster than it should because the model only has to copy, and the copying is useless at generation time, when the next character isn't available. The excerpt is the opening of tinyshakespeare, the same public-domain text nanoGPT trains on.[3]

Production models don't use characters. GPT-2 and its successors use byte-level BPE. It starts from the 256 possible byte values, merges the most frequent adjacent pair into a new token, and repeats until the vocabulary reaches its target size, 50,257 for GPT-2.[5] A regular expression first splits the text so that merges never cross a boundary between letters and digits or letters and punctuation. The result can be decoded back to the exact text, and on average each token covers about four bytes.[6][5]

Nothing downstream changes: the model sees integer ids either way. What changes is the unit of the loss. Each BPE prediction covers about four characters, so a loss per token and a character model's loss per character can only be compared after converting both to bits per byte.

Worked example

GPT-2 124M trained by nanoGPT reaches about 2.85 nats per token. 2.85 / ln 2 = 4.11 bits per token. At four bytes per token, that's about 1.03 bits per byte.

This page's model ends near 2.39 nats per character, which is 3.44 bits per character. Its text is plain ASCII, one byte per character, so that's 3.44 bits per byte.

The two runs score different text, so the numbers illustrate the unit conversion and say nothing about which model is better.
