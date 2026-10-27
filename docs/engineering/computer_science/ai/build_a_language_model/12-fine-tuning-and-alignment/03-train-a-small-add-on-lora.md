# Train a small add-on: LoRA

Full fine-tuning updates every weight, and that's expensive. The optimizer keeps extra numbers for every weight it trains: about 16 bytes per parameter in mixed precision, roughly 112 GB for a 7-billion-parameter model (the capstone page breaks that count down). And each task needs its own full copy of the model.

LoRA (low-rank adaptation) freezes the pretrained weight matrix and trains a small correction next to it. The correction is the product of two thin matrices, A and B. Being thin, they hold far fewer numbers than the full matrix, and their product can change it in at most r independent directions, where r, the rank, is the thin side's size. (Rank is the number of independent directions a matrix uses; Linear Algebra for ML covers it.)[7]

Worked example

A 4096 × 4096 projection holds 4096 × 4096 = 16,777,216 weights.

A rank-16 adapter has A of 4096 × 16 and B of 16 × 4096: 16 × (4096 + 4096) = 131,072 trainable numbers, about 0.78% of the matrix.

The reference implementation starts A with the same random draw an ordinary linear layer gets and starts B at zero. The product is then zero, so the adapted model begins exactly where the pretrained one is. The product is scaled by α/r, where α is a constant you choose.[7] After training, the product is added into the frozen weight once and the adapter disappears, so the merged model runs exactly as fast as the original.[8]

The first formula below is the adapted weight. The other two are the adapter's gradients, which show how cheaply backpropagation reaches A and B:
W = W0 + (α/r)·A·B ∂L/∂A = (α/r)·∂L/∂W·Bᵀ ∂L/∂B = (α/r)·Aᵀ·∂L/∂W

In words: the weight the model uses is the frozen original plus a scaled product of the two thin matrices. Each factor's gradient is the gradient the full matrix would have received anyway, multiplied by the other factor, so a backward pass that already computes ∂L/∂W needs only two small extra matrix products. The adapter has r·(in + out) parameters instead of in·out.

PEFT, Hugging Face's adapter library, also offers rsLoRA, which scales by α/√r instead.[9]
Teach the farm model about spacecraft: full fine-tuning against LoRA
LoRA rank r
Learning rate
Step
Play
Reset
Base model: 4160 parameters, farm validation loss 1.099, space 3.690 before fine-tuning. After 0 iterations on the space corpus at rate 0.01: full fine-tuning (all 4160 parameters) reaches space 3.690 and farm 1.099; LoRA rank 4, α = 8 (1024 trainable parameters, 25%) reaches space 3.690 and farm 1.099.

Press Step to run ten more iterations, or Play to run all 300; change the rank or rate to start a new pair of runs. Both runs start from the same farm-trained weights and train on the same space-domain batches, and α is 2r, so the scale is always 2. Left: loss on held-out space text. Right: loss on held-out farm text, which shows forgetting. At rate 0.01, full fine-tuning takes the space loss from 3.69 to about 0.46 while the farm loss climbs from 1.10 to about 4.5, worse than the 3.50 a uniform guess over the 33 characters scores: the old domain is gone. LoRA at rank 4 trains 1,024 parameters on the four attention and MLP matrices of this 4,160-parameter model, reaches about 1.35 on space text and holds farm text near 2.5. Rank trades one against the other: rank 1 ends at 2.0 space and 2.1 farm, rank 16 at 0.86 and 3.0. A base model this small has little spare capacity, which is why the gap between full fine-tuning and LoRA here is far larger than in the LoRA README's results, where adapting 0.8M of 125M or 0.35M of 355M parameters matched or beat full fine-tuning.[8]
Check Your Understanding Q1 of 2

A LoRA adapter on a 4096 × 4096 projection with rank 8 trains how many parameters, and what does the model compute at inference after merging?
8 × (4096 + 4096) = 65,536 parameters; one 4096 × 4096 multiply, as before.
8 × 4096 parameters; two extra matrix multiplies at inference.
4096 × 4096 / 8 parameters; the same single multiply.
0/2 correct
