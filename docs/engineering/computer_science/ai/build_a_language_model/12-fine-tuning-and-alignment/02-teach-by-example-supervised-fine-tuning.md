# Teach by example: supervised fine-tuning

You want the model to answer "Q: the cat? A:" with "the mat." The direct way is to collect prompts paired with the answers you want and keep training on them with pretraining's loss. That's supervised fine-tuning (SFT). Each example's prompt and completion (the answer) are joined into one sequence and split into tokens. The loss at each position is the cross-entropy: minus the log of the probability the model gave the token that actually comes next.

One change matters: which positions count. If the loss counts every position, part of the training goes into predicting the question, which the user will always supply. So trainers usually count only the completion. By default TRL, Hugging Face's library of fine-tuning trainers, counts only completion tokens, and for conversations an option counts only the assistant's turns. Both work the same way: the target at every position that shouldn't count is set to an ignore index, which the loss and its gradient skip. PyTorch's cross-entropy uses −100 and nanoGPT uses −1.[5][6] Padding is masked the same way, and packing several short examples into one sequence keeps the batch full.[5]

Worked example

The prompt "Q: the cat? A: " has 15 characters and the completion "the mat.⏎" has 9, counting the newline. Each position predicts the next character, so the 24 characters give 23 targets.

The first 14 targets are prompt characters and the last 9 are the answer. With the prompt masked, those 14 targets are set to −1 and the loss averages over the 9 answer targets.

Without masking, 14 of the 23 targets, 61%, are spent on predicting the question.

The SFT loss averages the cross-entropy over the positions that count:
LSFT = −(1/|C|) Σt ∈ C log pθ(yt+1 | x, y≤t)

In words: for every position whose next token belongs to the completion, take minus the log of the probability the model gave that token, then average over those positions. C is the set of positions that count. Prompt tokens still go into the model as context; they just aren't graded, so they produce no gradient.
Fine-tune on question–answer pairs with and without prompt masking
Learning rate
Step
Play
Reset
Example: "Q: the fox? A: " (15 characters) then "the field.⏎" (11 characters). With the prompt masked, 11 of 25 target positions produce gradient (44%). After 0 iterations at rate 0.01: completion-token loss 1.285 with prompt masking, 1.285 without; the base model scored 1.285.

Press Step to run ten more training iterations (batch 8) on both models, or Play to keep going. The base model was pretrained on 7,200 characters of farm sentences; the fine-tuning examples look like "Q: the cat? A: the mat." with a fixed answer for each animal. The left panel compares a run that masks the prompt (purple) with one that doesn't (green), both scored only on answer characters. Both start at 1.29 nats, because the farm model already predicts "the " well. At the default rate of 0.01, the masked run reaches 0.21 after 20 iterations and 0.04 after 50; the unmasked run reaches 0.45 and 0.11 and stays higher at every later logged point, because more than half the targets in each of its batches are question characters. The right panel shows the masked model's loss on each character of one example: once trained, up to 11 nats on the question characters it never learned to predict, and close to zero on every answer character.
