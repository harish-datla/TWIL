# Sample from it

To write text, the model reads a prompt, takes its prediction for the next character, picks one, appends it and repeats. Once the text is longer than the 32-character window, only the last 32 characters are fed in.[7] Always taking the most likely character, greedy decoding, tends to get stuck in loops; drawing at random from the predicted probabilities gives variety. Two knobs reshape the probabilities before the draw.

Temperature divides the logits by a number before the softmax. Below 1 it sharpens the distribution toward the favorite; above 1 it flattens it toward uniform. Top-k keeps only the k most likely characters, sets the rest to zero and rescales the survivors to sum to 1. That cuts off the long tail of rare characters a flat distribution would otherwise sometimes pick.

Worked example

Logits (2, 1, 0) for three characters.

Temperature 1: (0.665, 0.245, 0.090). Temperature 0.5, logits become (4, 2, 0): (0.867, 0.117, 0.016). Temperature 2, logits become (1, 0.5, 0): (0.506, 0.307, 0.186).

Top-k with k = 2 at temperature 1 drops the third character and rescales: (0.731, 0.269, 0).
Reshape the next-character distribution
Prompt
Temperature
0.8
Top-k
Use current model
Model after 0 training iterations (untrained: run the training widget above). After "First Citizen:⏎" the most likely next character is "y" with probability 0.025; the distribution's entropy at temperature 0.8 is 4.05 nats (5.84 bits), so a sample is as unpredictable as a fair choice among 57.2 characters.

Pick a prompt and drag the temperature or change top-k. The left bars show the twelve most likely next characters; the right panel shows the greedy continuation in yellow, then one sampled continuation. Train the model above first: this widget reads whatever state the training widget has reached, and refreshes every ten steps of training and on any control change. After 900 iterations on the 16 KB text, the character after "First Citizen:⏎" is a capital letter with high probability, as it is in the text, and the entropy of the distribution falls from about 2.4 nats at temperature 0.8 to under 1 nat at 0.3. The greedy continuation, the most likely character at every step, falls into "the the the" or "an and the an" within a few words: once the most likely continuation of a state leads back to that state, there's no randomness to leave it. The Transformers generation documentation describes the same thing as greedy search repeating itself on longer outputs.[12] The sampled line below it is less fluent and doesn't loop.
