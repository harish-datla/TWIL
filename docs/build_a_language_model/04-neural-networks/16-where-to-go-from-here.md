# Where to go from here

The network built on this page is called a multilayer perceptron (MLP) or feedforward neural network. It is the base that later architectures build on, and fully connected MLP blocks still sit inside most of them. On its own it handles small tabular datasets and 2D toys, and struggles with the images, audio and text that made deep learning famous.

Continue with Automatic Differentiation from Scratch, which turns the hand-written backward pass built here into a tape that differentiates any program. The Build a Language Model hub maps the rest of the series: training at depth, evaluation, embeddings, transformers, a trained GPT, alignment and serving, with convolutional networks and diffusion as adjacent pages.
Images: convolutions

A 256×256 grayscale image has 65,536 pixels. Connecting every pixel to each of, say, 500 neurons in an MLP's first hidden layer takes about 33 million weights for that layer alone, and the network has to learn the same edge detector separately at every position. Convolutional neural networks (CNNs) exploit the image's structure: a small filter of shared weights slides over the image, producing a feature map that records where its pattern appears. The algorithm from this page (backprop on a chain of differentiable operations) still drives training; only the layer types change.
Sequences: attention

For text, audio, or any data that comes in a sequence, an MLP needs a fixed input size and has no built-in notion of order. The standard sequence models before 2017, recurrent networks, read one token at a time, so relating token i to token i+50 means carrying information through every position in between. Vaswani et al.'s 2017 Attention Is All You Need[9] replaces recurrence with scaled dot-product attention. Each token scores how relevant every other token is to it, turns the scores into weights with softmax, and takes a weighted mix of the other tokens' content, so every pair of positions connects directly:
scaled dot-product attention Attention(Q, K, V) = softmax( QK⊤√dk )V

In words: each token gets three vectors, a query Q (what it's looking for), a key K (what it offers) and a value V (its content). Take the dot product of every query with every key, divide by √dk to keep the numbers moderate, softmax each row into weights, and use them to mix the values. Each token ends up with its own weighted summary of the sequence.

Stack blocks of attention followed by a small MLP (with residual connections and normalization around each) and you have a transformer, the architecture behind nearly every major language model since 2018. Karpathy's nanoGPT[10] is a compact reference implementation of a GPT-style transformer: about 300 lines for the model, 300 for the training loop.
You don't need GPT-4 to do interesting things

Eldan and Li's TinyStories paper[11] showed that transformers with under 10 million parameters, some with a single transformer block, can write fluent, consistent short stories when trained on a synthetic dataset restricted to words a typical 3- to 4-year-old understands. The authors report that every model in the paper trains on a single V100 GPU in at most 30 hours, so the whole pipeline is within reach of a hobby budget.

Building a tiny transformer takes the same forward/backward/update loop with different layers in the middle. Karpathy's series[15] and Perivolaropoulos's Train Your Own LLM From Scratch workshop[19] both walk through one step by step.
You finished

You now have a code-level picture of how neural networks are trained: forward pass, loss, backpropagation, update. Larger models change the layer types, the optimizer and the scale, but they are trained by the same loop.
The final exam

Five questions covering the whole tutorial. If you can answer all five without scrolling back up, you've got the fundamentals.
Check Your Understanding Q1 of 5

You train a 2-class network with softmax + cross-entropy. After one example with true class 0, the prediction is [0.7, 0.3]. What is the gradient of the loss with respect to the output logits?
[−0.3, 0.3]
[0.7, 0.3]
[−0.7, −0.3]
0/5 correct
