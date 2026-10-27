# Train it

Training repeats one short loop 900 times: draw a batch, run the forward pass, run the backward pass, nudge every weight. It's nanoGPT's loop at toy scale.[4] Two safety measures and a schedule sit around the update.

    Clip the gradient. Measure the gradient's overall size, its global norm: the square root of the sum of every squared gradient entry. If it's above 1, scale the whole gradient down to norm 1, so one unlucky batch can't throw the weights far.
    Update with AdamW. Adam gives each weight its own step size, using running averages of that weight's recent gradients and squared gradients. AdamW adds weight decay: every step it shrinks each matrix toward zero by a small fraction, here with decay 0.1 on the matrices and none on the LayerNorm gains.[4][10]
    Schedule the learning rate. The rate climbs in a straight line from near 0 to its peak over the first 50 iterations (warmup), then follows a cosine curve down to a tenth of the peak by the last iteration.

Worked example

Peak rate 0.01 over 900 iterations. Iteration 0: 0.0002. Iteration 24: 0.0049. Iteration 50: 0.01. Halfway through the decay, iteration 475: 0.0055. Iteration 900: 0.001.

At the peak, weight decay multiplies every matrix by 1 − 0.01 × 0.1 = 0.999 each step. A batch whose gradient has global norm 4 gets every gradient entry multiplied by 0.25.

Adam's β₂ sets how long its running average of squared gradients remembers: roughly the last 1/(1 − β₂) steps. nanoGPT's default of 0.95 remembers about 20 steps. This page uses 0.99, about 100 steps, because each iteration sees only 256 tokens and an average over so few is noisy. nanoGPT's character-level config makes the same choice for the same reason.[11] Validation loss is the mean over two fixed batches of the held-out text, 512 characters, so it's noisy to a few hundredths. nanoGPT averages 200 batches for the same reason.[4]
Train the one-block GPT in the browser
Training text
Peak learning rate
Step
Play
Reset
15264 parameters, 14361 training tokens, batch 8×32, peak rate 0.01. After 0 iterations: train loss 4.070, validation loss 4.070 (perplexity 58.54, 5.87 bits per character; best so far 4.070). Yardsticks on the same validation split: uniform ln 58 = 4.060, smoothed unigram model 3.215, smoothed bigram model 2.615 nats.

Press Play to train for 900 iterations, or Step to run six at a time. The left panel plots training loss (purple) and validation loss (green) against the three grey yardsticks; the right panel shows text sampled from the model so far. Each Step runs six iterations of batch 8 × 32 on this tab's CPU. The loss starts at ln 58 = 4.06, the uniform baseline, which confirms the initialization. On the 16 KB excerpt at peak rate 0.01 the validation loss passes the unigram line within 30 iterations, crosses the bigram model's 2.615 around iteration 370 and ends near 2.39, so the model has learned more than one character of context. Switch to the first 2 KB and the curves separate: training loss keeps falling, to about 0.6, while validation loss bottoms out near 2.6 after about 115 iterations, turns upward and ends above 5, worse than the uniform model, because a model that has memorized its training text is confidently wrong on text it hasn't seen. The samples are drawn at temperature 0.8, starting from a newline.

Two checks that cost nothing catch many common training bugs.[2] First, the loss at initialization should be −ln(1/V) = ln V to within a few hundredths, which this model shows on its first render. A value far above it means the initial logits are too large. A value clearly below it means something is handing the model the answer: feeding this tied-embedding model its targets as inputs drops the initial loss from 4.07 to 3.66 before any training. Second, a small model should be able to drive the loss on a small fixed set of text close to zero. The 2 KB run above is a slow version of that overfitting test: a model with 15,000 parameters that can't memorize 1,800 characters has a bug.
Check Your Understanding Q1 of 2

The loss on the first render is 4.06 for a 58-character vocabulary. What does that establish?
The initialization is broken: the model should already be better than uniform.
The logits start near zero, so the model predicts uniformly, which is the intended starting point.
The data is random and cannot be modelled.
0/2 correct
