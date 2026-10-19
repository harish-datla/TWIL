# A causal attention kernel with a cache offset

These complete programs implement one attention head on 2-number queries, keys and values. They apply the causal mask, with a cache offset, before the softmax, and they subtract the largest allowed score before exponentiating, so exp never sees a large positive number. Each program then checks three things: the first row, which may see only one key, returns that key's value; a cached query matches the full computation; and changing a future value leaves an earlier row unchanged.
Stable causal attention, complete programs
C++20
Rust 2021

#include <algorithm>
#include <array>
#include <cassert>
#include <cmath>
#include <vector>
using Vector = std::array<double,2>;
using Sequence = std::vector<Vector>;
Sequence attention(const Sequence& queries, const Sequence& keys,
                   const Sequence& values, int queryOffset = 0) {
    assert(keys.size() == values.size() && !keys.empty());
    Sequence output(queries.size(), Vector{0,0});
    for (int row = 0; row < static_cast<int>(queries.size()); ++row) {
        // queryOffset is the number of cached tokens before this query chunk.
        const int lastKey = row + queryOffset;
        assert(lastKey >= 0 && lastKey < static_cast<int>(keys.size()));
        std::vector<double> scores(lastKey+1);
        for (int column = 0; column <= lastKey; ++column)
            scores[column] = (queries[row][0]*keys[column][0]
                            + queries[row][1]*keys[column][1])/std::sqrt(2.0);
        const double maximum = *std::max_element(scores.begin(), scores.end());
        double denominator = 0;
        for (double& score : scores) {
            score = std::exp(score-maximum); // Prevent large positive exponentials.
            denominator += score;
        }
        for (int column = 0; column <= lastKey; ++column)
        for (int feature = 0; feature < 2; ++feature)
            output[row][feature] += scores[column]/denominator * values[column][feature];
    }
    return output;
}
int main() {
    const Sequence queries{{1,0},{0,1},{1,1}};
    Sequence values{{2,0},{0,3},{-1,1}};
    const auto full = attention(queries,queries,values);
    assert(std::abs(full[0][0]-2.0) < 1e-12); // First token has one legal key.
    const auto cached = attention(Sequence{queries.back()},queries,values,2);
    for (int feature = 0; feature < 2; ++feature)
        assert(std::abs(full.back()[feature]-cached[0][feature]) < 1e-12);
    values.back() = {100,100};
    const auto changed = attention(queries,queries,values);
    assert(full[1] == changed[1]); // Future values cannot influence row one.
}

This reference kernel leaves out learned projection matrices, multiple heads, position encoding, padding masks, batching and fast matrix multiplication. It defines the mixing step and nothing more. The caller must pass finite vectors whose scores stay finite, and every row needs at least one allowed key: subtracting the maximum can't rescue a dot product that has already overflowed. A complete decoder also needs a token embedding, a stack of blocks, and an output projection onto the vocabulary, trained to predict the shifted targets.

When you build the block stack, keep tensor shapes and the normalization order exactly as the reference model has them. Apply the position encoding to new queries and keys using their absolute positions, store keys in the same form the reference pass uses (after rotation, if that's where it stores them), and compare the whole decoded output against an uncached run. A cached implementation can return plausible numbers while attending to the wrong tokens.
Check Your Understanding Q1 of 2

A one-token cached query follows 12 old tokens. How many keys may it attend to?
One, because the local query row is zero.
12, because causal attention excludes the current position.
13, including its new key.
0/2 correct
