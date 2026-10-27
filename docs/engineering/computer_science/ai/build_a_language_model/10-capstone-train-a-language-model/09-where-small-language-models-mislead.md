# Where small language models mislead
Perplexities are only comparable at the same tokenization on the same text

A character model's 2.4 nats and a BPE model's 2.85 nats are in different units: one is per character, the other per token of roughly four characters. Convert both to bits per byte before comparing, and compare only on the same evaluation text. The Evaluation page covers how the score also depends on context length and stride.

A falling training loss on its own isn't evidence of learning. The 2 KB run reaches a lower training loss than the 16 KB run, yet it's a worse model of the language on every held-out measure. Report validation loss, say how noisy it is, and look at samples.

Greedy samples make a model look worse than it is. A model whose most-likely-character output loops can still assign sensible probabilities; the loop comes from the decoding rule.[12] Judge a language model by its loss and by samples at a stated temperature, not by its greedy output alone.

The scaling rules of thumb leave things out. The 6 FLOPs per parameter per token count omits attention, which matters once the context T is large compared with the width C. The 16 bytes per parameter omits activations, the intermediate values the backward pass needs, which dominate at large batch and context. Treat the widget's memory figure as a floor, and its training time as only as good as the utilization you assume.
