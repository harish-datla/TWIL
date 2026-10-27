# A stable cross-entropy, checked

The programs below, one in C++ and one in Rust, implement the pieces from this page: log-sum-exp with the maximum subtracted, log-softmax, cross-entropy against a one-hot target, entropy and KL divergence. Their assertions check the claims made above. The softmax doesn't change when 1000 is added to every logit, a shift that would overflow a naive exponential. CE = H + KL holds on a fixed pair of distributions. KL is positive between different distributions, zero from a distribution to itself, and different in the two directions. The gradient p − y matches a finite-difference estimate of the loss's slope.
Log-sum-exp, cross-entropy, entropy, KL: complete programs
C++20
Rust 2021

#include <algorithm>
#include <cassert>
#include <cmath>
#include <cstddef>
#include <vector>
using Vector = std::vector<double>;
double logSumExp(const Vector& logits) {
    const double maximum = *std::max_element(logits.begin(), logits.end());
    double total = 0;
    for (double logit : logits) total += std::exp(logit - maximum); // Every term is at most 1: no overflow.
    return maximum + std::log(total);
}
Vector logSoftmax(const Vector& logits) {
    const double normalizer = logSumExp(logits);
    Vector logProbabilities(logits.size());
    for (std::size_t index = 0; index < logits.size(); ++index) logProbabilities[index] = logits[index] - normalizer;
    return logProbabilities;
}
// Cross-entropy of a one-hot target against the softmax of the logits: −log p[target].
double crossEntropyFromLogits(const Vector& logits, std::size_t target) { return -logSoftmax(logits)[target]; }
double entropyBits(const Vector& probabilities) {
    double sum = 0;
    for (double p : probabilities) if (p > 0) sum -= p * std::log2(p); // 0 log 0 counts as 0.
    return sum;
}
double crossEntropyBits(const Vector& truth, const Vector& model) {
    double sum = 0;
    for (std::size_t index = 0; index < truth.size(); ++index)
        if (truth[index] > 0) sum -= truth[index] * std::log2(model[index]); // Infinite if model[index] == 0.
    return sum;
}
double klBits(const Vector& truth, const Vector& model) { return crossEntropyBits(truth, model) - entropyBits(truth); }
int main() {
    const Vector logits{2.0, 1.0, -1.0};
    Vector shifted = logits;
    for (double& logit : shifted) logit += 1000.0; // exp(1002) overflows a double; the stable form does not care.
    const Vector probabilities = logSoftmax(logits), shiftedProbabilities = logSoftmax(shifted);
    double mass = 0;
    for (std::size_t index = 0; index < 3; ++index) {
        assert(std::abs(probabilities[index] - shiftedProbabilities[index]) < 1e-9); // Shift invariance.
        mass += std::exp(probabilities[index]);
    }
    assert(std::abs(mass - 1.0) < 1e-12); // The probabilities sum to one.
    const Vector truth{0.7, 0.2, 0.1}, model{0.5, 0.3, 0.2};
    assert(std::abs(crossEntropyBits(truth, model) - (entropyBits(truth) + klBits(truth, model))) < 1e-12);
    assert(klBits(truth, model) > 0 && std::abs(klBits(truth, truth)) < 1e-12); // Gibbs' inequality.
    assert(std::abs(klBits(truth, model) - klBits(model, truth)) > 1e-3);    // Not symmetric.
    const std::size_t target = 1;
    for (std::size_t index = 0; index < 3; ++index) {
        Vector plus = logits, minus = logits;
        plus[index] += 1e-5; minus[index] -= 1e-5;
        const double numerical = (crossEntropyFromLogits(plus, target) - crossEntropyFromLogits(minus, target)) / 2e-5;
        const double analytic = std::exp(probabilities[index]) - (index == target ? 1.0 : 0.0); // p − y.
        assert(std::abs(numerical - analytic) < 1e-7);
    }
}

What's intentionally missing

Batching over many examples, class weights, label smoothing, an ignore index for padding, and the fused kernels that compute the loss and its gradient in one pass over the logits. Production cross-entropy for a language model also avoids materializing the full vocabulary-sized probability vector for every token when it can.
