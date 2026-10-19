# Attention: each position gathers from the others

In "the animal was tired, so it lay down", the vector for "it" needs information from "animal", several tokens back. A dense layer can't move it there: it transforms each position's vector on its own. Attention is the layer that moves information between positions, and it decides how much to take from each position based on the input itself.

Every position produces three vectors from its hidden representation, each through its own learned projection (a matrix multiply):

    A query: what this position is looking for.
    A key: what this position offers to be matched against.
    A value: the content this position hands over if it's chosen.

For one position, take the dot product of its query with every key. A high score means a good match. A softmax turns the row of scores into weights that are positive and sum to 1, and the output is the weighted average of the values.[2]

Worked example

Three tokens, "the cat sat", with 2-number vectors. The query for "sat" is (2, 0). The keys are the = (0, 1), cat = (2, 0), sat = (1, 1), and the values are the = (0, 0), cat = (10, 0), sat = (0, 10).

Scores: (2, 0) · each key = 0, 4, 2. Divide by √2 (the key width is 2): 0, 2.83, 1.41.

Softmax: weights 0.045, 0.768, 0.187. Output: 0.768 × (10, 0) + 0.187 × (0, 10) = (7.68, 1.87), mostly the value of "cat".

Done for all positions at once, with the vectors stacked as rows of matrices, that's the whole operation:
Q = XWQ;  K = XWK;  V = XWV
A = softmax(QKᵀ / √dk)
Y = AV

In words: project every token into a query, a key and a value. Score every query against every key, scale the scores down by √dk, and softmax each row into weights. Each output row is the weighted average of the value rows. With N tokens, A is an N×N grid.

Why divide by √dk? A dot product adds up dk products, so it grows with the vector width. If the entries were independent random numbers with mean 0 and variance 1, the dot product would have variance dk. Scores that large push the softmax to put nearly all its weight on one key, and the other keys then get almost no gradient to learn from. Dividing by √dk brings the spread back to about 1.[2] Real learned vectors don't follow those assumptions exactly; the argument explains the size of the factor.

Worked example

dk = 64, so random dot products have a standard deviation of √64 = 8, and a score gap of 16 is unremarkable. A softmax over (16, 0) gives the first key 0.9999999. Divided by 8 the scores are (2, 0), and the softmax gives 0.88 and 0.12.
Change one query, recompute the mixture
Query angle (degrees)
35
Query length
2
Reset
Scores = dot(query,key)/√2. Weights [0.514, 0.363, 0.051, 0.072]; sum 1.000000; mixed value [1.049, 0.263].

Turn the query with the angle slider, stretch it with the length slider, and read the bars. The four keys point along the coordinate axes, and the values are separate, fixed vectors. Purple bars are the attention weights; the right panel draws the four values and the mixed output in pink. At the default (35°, length 2) the weights are 0.514, 0.363, 0.051 and 0.072, so the output leans toward the first value. A longer query scales every score up, which concentrates the weight on the best-matching key.

The names query, key and value describe each vector's role in the calculation. Training doesn't have to give them any meaning a person would recognize. Attention weights aren't probabilities that something is true. They're the coefficients of a weighted average: a large weight just means that position's value counts for more in this output.

Multi-head attention runs several of these mixtures side by side, each with its own projections and a smaller width, then concatenates their outputs and applies one more projection. The total width can stay the same as with a single head. Different heads can learn different relations, but nothing in the architecture guarantees a head a fixed human-readable job such as "the grammar head".[2]
Check Your Understanding Q1 of 2

Which values does softmax normalize together?
All rows and columns in one global distribution.
The coordinates of every value vector.
The key scores for one query row.
0/2 correct
