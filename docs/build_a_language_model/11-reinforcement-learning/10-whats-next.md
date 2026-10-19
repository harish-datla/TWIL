# What's next

Fine-tuning and Alignment applies the policy gradient from section 07 to a language model. It covers a reward model learned from people's choices between two replies, and RLHF, which maximizes that reward while a KL penalty keeps the model close to where it started (the penalty grows as the model's probabilities drift from the original's). It ends with DPO, which reaches the same optimum without a separate reward model.
