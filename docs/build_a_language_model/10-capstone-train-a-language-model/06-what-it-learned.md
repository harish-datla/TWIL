# What it learned

A final loss of 2.39 measures how much the model learned. Two further measurements show what it learned. The first shows where each attention head looks on a fixed prompt. The second splits the loss by how many characters of context each position had, which shows how far back the model actually reads.

Attention patterns need a baseline too. A head that spread its weight evenly would give the 5th character's predecessor 1/5 of its attention and the 32nd character's predecessor 1/32. A head that has learned to read the previous character gives it much more.

Worked example

Even attention over a 32-character prompt: position t can see t characters, so it gives 1/t to the one just before it. Averaged over positions 2 to 32, that's 0.10, the uniform figure the caption below compares the trained heads against.
Attention weights of one head on a fixed prompt
Head
Prompt
Use current model
Head 0 after 0 iterations, 32 characters. Rows are query positions, columns the positions they read; the upper triangle is masked. Strongest source for the last three characters: "d"→"e" at 20 (0.03), "e"→"o" at 18 (0.03), "e"→"n" at 12 (0.03). Mean weight on the immediately preceding character: 0.10 (uniform attention would give 0.10).

Pick a head and a prompt. Rows are positions in the prompt and columns are the positions each one reads from; brighter cells carry more weight, and outlines mark the strongest source for each of the last three characters. Before training every row is nearly flat, each position spreading its weight over all the positions it may see. After 900 iterations the heads differ: in the run above, the mean weight a position gives to the character immediately before it rises to between 0.17 and 0.29 across the four heads, against 0.10 for uniform attention over a 32-character prompt, and the last few rows of head 0 put a quarter to a half of their weight on one recent character. This model, with one block and 8-wide heads, doesn't show the sharp previous-token and induction patterns reported in larger transformers; what it shows is the start of the first kind. Patterns depend on the random seed and the learning rate, so the specific rows describe this run only.
Loss by how much context the position has
Window start every
Use current model
Model after 0 iterations; 195 validation windows of 32 characters starting every 8 characters. Loss when the model sees 1 character 4.077, 2 characters 4.092, 4 characters 4.074, 8 characters 4.057, 32 characters 4.068; mean 4.070. Smoothed n-gram models on the same split: unigram 3.215, bigram 2.615; uniform 4.060 nats.

Change how often a validation window starts to score more or fewer windows. The line is the validation loss at each position of a 32-character window: position 1 is the loss when the model has seen one character, position 32 when it has seen the whole window, each averaged across windows. Untrained, the curve sits on the uniform line at 4.06. After 900 iterations at rate 0.01, the loss with one character of context is about 2.76, close to the bigram model's 2.615; from two characters onward it fluctuates between about 2.3 and 2.5 with no further trend. The model has learned to use roughly two or three characters of history and nothing beyond, which fits its samples: plausible fragments, but no words that depend on what came a line earlier.
