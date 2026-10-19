# A vector is a list that means something

A language model can't work with the word "cat" directly. It looks the word up in a table and gets back a list of numbers, 768 of them in GPT-2 small, called an embedding. Training nudges those numbers until words used in similar ways end up with similar lists. In math, a list of numbers like this is a vector, and the first thing a model needs is a way to measure how similar two vectors are.

The tool for that is the dot product: multiply the matching entries of two vectors and add up the results. That's all it is.

Worked example

a = (2, 1) and b = (1, 3).

a · b = 2×1 + 1×3 = 5.

In code: for (i = 0; i < n; ++i) sum += a[i] * b[i];

Draw each vector as an arrow from the origin and the result has a geometric meaning. The dot product is large and positive when the arrows point the same way, zero when they're at right angles, and negative when they point in opposite directions. It also grows with the arrows' lengths. A vector's length, its norm, comes from the same operation: √(a · a), Pythagoras' theorem for any number of entries. For a = (2, 1) that's √5 ≈ 2.24.
a · b = Σi ai bi = ‖a‖ ‖b‖ cos θ

In words: multiply matching entries and add them up. The result always equals the length of a, times the length of b, times the cosine of the angle between them.[1] That second form is why the dot product measures direction.

For the example, ‖a‖‖b‖ = √5 × √10 ≈ 7.07, so cos θ = 5 / 7.07 ≈ 0.71 and the angle is 45°. You can't picture an angle between two 768-number vectors, but the arithmetic is identical, so "angle" still means "how closely do these point the same way".

Often you want direction only, without length. Divide the dot product by both lengths and you get cosine similarity, a score from −1 to 1. It's the standard way to compare embeddings, because a long vector and a short one pointing the same way should count as alike.[2] The dot product itself shows up everywhere else in a model:

    A neuron computes w · x + bias: a dot product of its weights with its input. Each weight says how much one input matters.
    Attention, the core of a transformer, scores how relevant each earlier word is by taking a dot product between a "query" vector and a "key" vector for every pair of words.
    Vector search ranks documents by the cosine similarity of their embeddings to the question's embedding.

Turn one vector, watch the dot product and the angle
Angle of b (degrees)
30
Length of b
1.6
Reset
a = (2, 1), b = (1.386, 0.800). a·b = 2×1.386 + 1×0.800 = 3.5713. ‖a‖ = 2.2361, ‖b‖ = 1.6000, cos θ = 0.9982; angle between them 3.4°. Projection of b onto a has signed length 1.5971.

The purple arrow a = (2, 1) stays put; drag the angle to turn b. The dashed arrow is the shadow b casts on a (its projection), and its signed length is a·b/‖a‖. The dot product flips sign as the angle crosses 90°. Dragging the length slider scales a·b up and down but leaves cos θ unchanged, which is what makes cosine similarity ignore length.
Check Your Understanding Q1 of 2

Two embeddings have cosine similarity 0.98, but one is twice as long as the other. What does that tell you?
Their dot product is also about 0.98.
They point in almost the same direction; the dot product also grows with both lengths.
Their dot product must be negative because the lengths differ.
0/2 correct
