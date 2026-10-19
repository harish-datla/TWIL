# Two checked programs

The first program puts a rank-2 adapter on an 8 × 6 linear layer. It works out the adapter's gradients from the full-weight gradient and checks all 28 of them against centered finite differences: nudge each number up and down slightly and measure how the loss changes. It also asserts that the merged weight gives the same output as running the frozen layer and the adapter side by side. The second program builds the preference pipeline over six responses: 2,000 comparisons sampled through Bradley–Terry from hidden rewards, a reward model fit by gradient descent with the centering penalty, the closed-form KL-regularized optimum, an exact policy gradient that reaches it, and DPO reaching the same policy from the raw pairs. Both run in well under a second.
LoRA on one linear layer: gradients checked, merge verified
C++20
Rust 2021

// LoRA on one linear layer: the adapter's gradients, checked against finite differences, and the merge identity.
// Build: g++ -std=c++20 -O2 -Wall -Wextra -Wpedantic -Werror lora.cpp && ./a.out
#include <cassert>
#include <cmath>
#include <cstdio>
#include <vector>

struct Rng {  // xorshift32 with a fixed seed
    unsigned state = 11;
    double next() { state ^= state << 13; state ^= state >> 17; state ^= state << 5; return state / 4294967296.0; }
};

constexpr int kIn = 8, kOut = 6, kRank = 2, kRows = 4;
constexpr double kAlpha = 4.0, kScaling = kAlpha / kRank;  // loralib: scaling = lora_alpha / r

// y[rows][out] = x[rows][in] · W[in][out]
std::vector<double> matmul(const std::vector<double>& left, const std::vector<double>& right, int rows, int inner, int columns) {
    std::vector<double> out(std::size_t(rows) * columns, 0.0);
    for (int row = 0; row < rows; ++row) for (int i = 0; i < inner; ++i) for (int c = 0; c < columns; ++c) out[row * columns + c] += left[row * inner + i] * right[i * columns + c];
    return out;
}
// The adapted weight the model actually multiplies by: W₀ + scaling · A · B.
std::vector<double> mergedWeight(const std::vector<double>& base, const std::vector<double>& A, const std::vector<double>& B) {
    std::vector<double> product = matmul(A, B, kIn, kRank, kOut);
    std::vector<double> merged(base);
    for (std::size_t i = 0; i < merged.size(); ++i) merged[i] += kScaling * product[i];
    return merged;
}
// Half the squared error between x·W and the targets, summed over the batch.
double loss(const std::vector<double>& x, const std::vector<double>& weight, const std::vector<double>& targets) {
    std::vector<double> y = matmul(x, weight, kRows, kIn, kOut);
    double total = 0;
    for (std::size_t i = 0; i < y.size(); ++i) total += 0.5 * (y[i] - targets[i]) * (y[i] - targets[i]);
    return total;
}

int main() {
    Rng rng;
    auto uniform = [&](std::size_t size, double bound) { std::vector<double> values(size); for (double& value : values) value = (rng.next() * 2 - 1) * bound; return values; };
    std::vector<double> x = uniform(kRows * kIn, 1.0), targets = uniform(kRows * kOut, 1.0), base = uniform(kIn * kOut, 0.5);
    // loralib initialization: A ~ kaiming_uniform(a=√5), which is uniform in ±1/√in; B = 0, so the adapted layer starts equal to the base layer.
    std::vector<double> A = uniform(kIn * kRank, 1.0 / std::sqrt(double(kIn))), B(kRank * kOut, 0.0);
    assert(mergedWeight(base, A, B) == base);
    // Give B some content so the gradient with respect to A is not identically zero.
    for (double& value : B) value = (rng.next() * 2 - 1) * 0.3;

    // Forward through the merged weight, then the gradient of the loss with respect to that weight: xᵀ · (y − t).
    std::vector<double> merged = mergedWeight(base, A, B), y = matmul(x, merged, kRows, kIn, kOut);
    std::vector<double> weightGrad(kIn * kOut, 0.0);
    for (int row = 0; row < kRows; ++row) for (int i = 0; i < kIn; ++i) for (int o = 0; o < kOut; ++o) weightGrad[i * kOut + o] += x[row * kIn + i] * (y[row * kOut + o] - targets[row * kOut + o]);
    // Chain rule through W = W₀ + s·A·B: ∂L/∂A = s · ∂L/∂W · Bᵀ, ∂L/∂B = s · Aᵀ · ∂L/∂W. W₀ gets no update at all.
    std::vector<double> gradA(kIn * kRank, 0.0), gradB(kRank * kOut, 0.0);
    for (int i = 0; i < kIn; ++i) for (int o = 0; o < kOut; ++o) for (int k = 0; k < kRank; ++k) {
        gradA[i * kRank + k] += kScaling * weightGrad[i * kOut + o] * B[k * kOut + o];
        gradB[k * kOut + o] += kScaling * A[i * kRank + k] * weightGrad[i * kOut + o];
    }

    // Check every adapter entry against a centered finite difference of the loss.
    double worst = 0;
    auto check = [&](std::vector<double>& values, const std::vector<double>& analytic) {
        for (std::size_t i = 0; i < values.size(); ++i) {
            const double original = values[i], h = 1e-6;
            values[i] = original + h; const double plus = loss(x, mergedWeight(base, A, B), targets);
            values[i] = original - h; const double minus = loss(x, mergedWeight(base, A, B), targets);
            values[i] = original;
            const double numeric = (plus - minus) / (2 * h);
            worst = std::max(worst, std::fabs(numeric - analytic[i]) / std::max(1e-9, std::fabs(numeric) + std::fabs(analytic[i])));
        }
    };
    check(A, gradA); check(B, gradB);
    std::printf("adapter gradient check: worst relative error %.2e over %zu entries\n", worst, A.size() + B.size());
    assert(worst < 1e-6);

    // Merge identity: the two-path forward x·W₀ + s·(x·A)·B equals the single merged multiply, so inference pays nothing extra after merging.
    std::vector<double> twoPath = matmul(x, base, kRows, kIn, kOut), inner = matmul(x, A, kRows, kIn, kRank), outer = matmul(inner, B, kRows, kRank, kOut);
    double largestGap = 0;
    for (std::size_t i = 0; i < twoPath.size(); ++i) largestGap = std::max(largestGap, std::fabs(twoPath[i] + kScaling * outer[i] - y[i]));
    assert(largestGap < 1e-12);
    std::printf("trainable: %d adapter values versus %d in the full weight (%.0f%%); merge gap %.1e\n", kRank * (kIn + kOut), kIn * kOut, 100.0 * kRank * (kIn + kOut) / (kIn * kOut), largestGap);
    return 0;
}

Reward model, closed-form optimum, policy gradient and DPO on six responses
C++20
Rust 2021

// Preference optimization over a finite set of responses: a Bradley–Terry reward model, the KL-regularized
// optimum in closed form, exact policy-gradient ascent toward it, and DPO reaching the same policy from the pairs.
// Build: g++ -std=c++20 -O2 -Wall -Wextra -Wpedantic -Werror preference.cpp && ./a.out
#include <algorithm>
#include <cassert>
#include <cmath>
#include <cstdio>
#include <vector>

struct Rng {  // xorshift32 with a fixed seed
    unsigned state = 5;
    double next() { state ^= state << 13; state ^= state >> 17; state ^= state << 5; return state / 4294967296.0; }
};
struct Comparison { int chosen, rejected; };

double sigmoid(double value) { return 1 / (1 + std::exp(-value)); }
std::vector<double> softmax(const std::vector<double>& logits) {
    const double maximum = *std::max_element(logits.begin(), logits.end());
    std::vector<double> out(logits.size()); double total = 0;
    for (std::size_t i = 0; i < logits.size(); ++i) { out[i] = std::exp(logits[i] - maximum); total += out[i]; }
    for (double& value : out) value /= total;
    return out;
}
// Labels drawn through the Bradley–Terry model from the true rewards: P(i preferred over j) = σ(r_i − r_j).
std::vector<Comparison> sampleComparisons(const std::vector<double>& rewards, int count, Rng& rng) {
    std::vector<Comparison> pairs;
    while (int(pairs.size()) < count) {
        const int first = int(rng.next() * rewards.size()), second = int(rng.next() * rewards.size());
        if (first == second) continue;
        const bool firstWins = rng.next() < sigmoid(rewards[first] - rewards[second]);
        pairs.push_back({firstWins ? first : second, firstWins ? second : first});
    }
    return pairs;
}
double expectedValue(const std::vector<double>& policy, const std::vector<double>& values) {
    double total = 0; for (std::size_t i = 0; i < policy.size(); ++i) total += policy[i] * values[i]; return total;
}
double klDivergence(const std::vector<double>& policy, const std::vector<double>& reference) {
    double total = 0; for (std::size_t i = 0; i < policy.size(); ++i) if (policy[i] > 0) total += policy[i] * std::log(policy[i] / reference[i]); return total;
}

int main() {
    // Six candidate responses to one prompt: the reference model's probabilities and the true (hidden) rewards.
    const std::vector<double> reference{0.40, 0.30, 0.15, 0.10, 0.04, 0.01}, trueRewards{0.0, 0.5, 1.0, 1.5, 2.0, 4.0};
    const double beta = 0.5;
    Rng rng;
    const std::vector<Comparison> training = sampleComparisons(trueRewards, 2000, rng), heldOut = sampleComparisons(trueRewards, 2000, rng);

    // 1. Reward model: gradient descent on −log σ(r_chosen − r_rejected) with a small penalty on the mean, which pins the constant Bradley–Terry cannot determine.
    std::vector<double> fitted(reference.size(), 0.0);
    for (int step = 0; step < 400; ++step) {
        std::vector<double> gradient(fitted.size(), 0.0);
        for (const Comparison& pair : training) {
            const double miss = 1 - sigmoid(fitted[pair.chosen] - fitted[pair.rejected]);  // 1 − P(label) under the current scores
            gradient[pair.chosen] -= miss / training.size(); gradient[pair.rejected] += miss / training.size();
        }
        double mean = 0; for (double value : fitted) mean += value / fitted.size();
        for (std::size_t i = 0; i < fitted.size(); ++i) fitted[i] -= 0.5 * (gradient[i] + 0.01 * 2 * mean / fitted.size());
    }
    int agree = 0; for (const Comparison& pair : heldOut) agree += fitted[pair.chosen] > fitted[pair.rejected];
    std::printf("reward model: held-out agreement %.3f\n", double(agree) / heldOut.size());
    assert(double(agree) / heldOut.size() > 0.7);
    for (std::size_t i = 1; i < fitted.size(); ++i) assert(fitted[i] > fitted[i - 1]);  // the true ordering is recovered

    // 2. The KL-regularized optimum in closed form: π*(y) ∝ π_ref(y) · exp(r(y) / β).
    std::vector<double> optimumLogits(reference.size());
    for (std::size_t i = 0; i < reference.size(); ++i) optimumLogits[i] = std::log(reference[i]) + fitted[i] / beta;
    const std::vector<double> optimum = softmax(optimumLogits);

    // 3. Exact policy gradient on E_π[r] − β·KL(π‖π_ref), starting from the reference: reaches the same policy.
    std::vector<double> logits(reference.size());
    for (std::size_t i = 0; i < reference.size(); ++i) logits[i] = std::log(reference[i]);
    for (int step = 0; step < 3000; ++step) {
        const std::vector<double> policy = softmax(logits);
        std::vector<double> score(policy.size());
        for (std::size_t i = 0; i < policy.size(); ++i) score[i] = fitted[i] - beta * std::log(policy[i] / reference[i]);
        const double baseline = expectedValue(policy, score);
        for (std::size_t i = 0; i < policy.size(); ++i) logits[i] += 0.5 * policy[i] * (score[i] - baseline);
    }
    const std::vector<double> ascended = softmax(logits);
    double gap = 0; for (std::size_t i = 0; i < ascended.size(); ++i) gap = std::max(gap, std::fabs(ascended[i] - optimum[i]));
    std::printf("policy gradient versus closed form: largest probability gap %.1e; KL to reference %.3f, expected reward %.3f\n", gap, klDivergence(ascended, reference), expectedValue(ascended, fitted));
    assert(gap < 1e-4);

    // 4. DPO: no reward model, no sampling. Each pair moves the chosen logit up and the rejected one down by β·σ(−β·margin),
    //    where the margin is the difference of log(π/π_ref); the softmax normalizer cancels in that difference.
    std::vector<double> dpoLogits(reference.size());
    for (std::size_t i = 0; i < reference.size(); ++i) dpoLogits[i] = std::log(reference[i]);
    for (int step = 0; step < 3000; ++step) {
        const std::vector<double> policy = softmax(dpoLogits);
        std::vector<double> gradient(policy.size(), 0.0);
        for (const Comparison& pair : training) {
            const double margin = std::log(policy[pair.chosen] / reference[pair.chosen]) - std::log(policy[pair.rejected] / reference[pair.rejected]);
            const double weight = beta * sigmoid(-beta * margin);
            gradient[pair.chosen] -= weight / training.size(); gradient[pair.rejected] += weight / training.size();
        }
        for (std::size_t i = 0; i < dpoLogits.size(); ++i) dpoLogits[i] -= 0.5 * gradient[i];
    }
    const std::vector<double> dpoPolicy = softmax(dpoLogits);
    double dpoGap = 0; for (std::size_t i = 0; i < dpoPolicy.size(); ++i) dpoGap = std::max(dpoGap, std::fabs(dpoPolicy[i] - optimum[i]));
    std::printf("DPO versus closed form: largest probability gap %.3f; implicit rewards", dpoGap);
    for (std::size_t i = 0; i < dpoPolicy.size(); ++i) std::printf(" %+.2f", beta * std::log(dpoPolicy[i] / reference[i]));  // β·log(π/π_ref): the reward DPO learned, up to a constant
    std::printf("\n");
    assert(dpoGap < 0.05);  // the same pairs, through DPO, give the same policy the two-stage pipeline gives
    return 0;
}

What's intentionally missing

A language model behind the reward and the policies. The second program's "completions" are six labelled items, so expectations and the normalizer Z are exact sums, where a real run has samples and per-token log-probabilities. Also left out: dropout on the adapter, the MergedLinear handling of fused QKV weights, PPO's value network, clipping and generalized advantage estimation, reward whitening, the adaptive KL controller, and length normalization of sequence log-probabilities. The losses and gradients are the ones the cited trainers compute.
