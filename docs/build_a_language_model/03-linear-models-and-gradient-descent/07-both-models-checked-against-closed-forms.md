# Both models, checked against closed forms

These complete programs fit a line by gradient descent and check that it lands on the closed-form answer (slope = covariance / variance). Then they check a yes-or-no classifier's hand-derived gradient against centered finite differences and train it by gradient descent. Both listings use the same names so you can read them side by side.
Linear and logistic regression by gradient descent, complete programs
C++20
Rust 2021

#include <cassert>
#include <cmath>
#include <cstddef>
#include <vector>
struct Example { double feature; double target; };
// Mean squared error of y ≈ weight * feature + bias, and its gradient.
struct Fit { double loss, gradientWeight, gradientBias; };
Fit squaredLoss(const std::vector<Example>& data, double weight, double bias) {
    Fit fit{0, 0, 0};
    for (const Example& example : data) {
        const double residual = weight * example.feature + bias - example.target;
        fit.loss += residual * residual / data.size();
        fit.gradientWeight += 2 * residual * example.feature / data.size(); // Residual pushed back through the feature.
        fit.gradientBias += 2 * residual / data.size();
    }
    return fit;
}
double sigmoid(double logit) { return logit >= 0 ? 1 / (1 + std::exp(-logit)) : std::exp(logit) / (1 + std::exp(logit)); }
// Mean cross-entropy of a Bernoulli label under p = sigmoid(weight * feature + bias).
Fit logisticLoss(const std::vector<Example>& data, double weight, double bias) {
    Fit fit{0, 0, 0};
    for (const Example& example : data) {
        const double logit = weight * example.feature + bias, probability = sigmoid(logit);
        // log(1 + e^-|z|) + max(0, ∓z) equals −log p or −log(1 − p) without overflow.
        const double softplus = std::log1p(std::exp(-std::abs(logit)));
        fit.loss += (example.target == 1 ? softplus + std::fmax(0.0, -logit) : softplus + std::fmax(0.0, logit)) / data.size();
        fit.gradientWeight += (probability - example.target) * example.feature / data.size(); // (p − y) x
        fit.gradientBias += (probability - example.target) / data.size();
    }
    return fit;
}
int main() {
    std::vector<Example> regression;
    for (int index = 0; index < 20; ++index) {
        const double feature = -2.0 + 0.2 * index;
        regression.push_back({feature, 1.5 * feature + 0.5 + 0.3 * std::sin(7.0 * index)}); // Deterministic "noise".
    }
    // Closed form for one feature: weight = cov(x, y) / var(x), bias = mean(y) − weight * mean(x).
    double meanFeature = 0, meanTarget = 0;
    for (const Example& example : regression) { meanFeature += example.feature / 20; meanTarget += example.target / 20; }
    double covariance = 0, variance = 0;
    for (const Example& example : regression) {
        covariance += (example.feature - meanFeature) * (example.target - meanTarget);
        variance += (example.feature - meanFeature) * (example.feature - meanFeature);
    }
    const double closedWeight = covariance / variance, closedBias = meanTarget - closedWeight * meanFeature;
    double weight = 0, bias = 0;
    for (int step = 0; step < 2000; ++step) { // Plain gradient descent at a safe learning rate.
        const Fit fit = squaredLoss(regression, weight, bias);
        weight -= 0.1 * fit.gradientWeight; bias -= 0.1 * fit.gradientBias;
    }
    assert(std::abs(weight - closedWeight) < 1e-6 && std::abs(bias - closedBias) < 1e-6); // Descent reaches the normal-equations optimum.
    std::vector<Example> classification;
    for (int index = 0; index < 20; ++index) {
        const double feature = -3.0 + 0.3 * index + 0.8 * std::sin(5.0 * index);
        classification.push_back({feature, feature + 0.6 * std::cos(3.0 * index) > 0 ? 1.0 : 0.0}); // Overlapping classes.
    }
    const double probeWeight = 0.7, probeBias = -0.2, step = 1e-5;
    const Fit analytic = logisticLoss(classification, probeWeight, probeBias);
    const double numericalWeight = (logisticLoss(classification, probeWeight + step, probeBias).loss - logisticLoss(classification, probeWeight - step, probeBias).loss) / (2 * step);
    const double numericalBias = (logisticLoss(classification, probeWeight, probeBias + step).loss - logisticLoss(classification, probeWeight, probeBias - step).loss) / (2 * step);
    assert(std::abs(numericalWeight - analytic.gradientWeight) < 1e-8 && std::abs(numericalBias - analytic.gradientBias) < 1e-8); // Gradient check.
    double classifierWeight = 0, classifierBias = 0;
    const double initialLoss = logisticLoss(classification, 0, 0).loss;
    for (int iteration = 0; iteration < 500; ++iteration) {
        const Fit fit = logisticLoss(classification, classifierWeight, classifierBias);
        classifierWeight -= 0.5 * fit.gradientWeight; classifierBias -= 0.5 * fit.gradientBias;
    }
    assert(std::abs(initialLoss - std::log(2.0)) < 1e-12);                               // All-zero weights predict 0.5: loss log 2.
    assert(logisticLoss(classification, classifierWeight, classifierBias).loss < 0.5 * initialLoss); // Training halved it.
}

What's intentionally missing

Multiple features and the matrix form, minibatching and shuffling, standardization computed from training statistics and reused at test time, a learning-rate schedule, weight decay, early stopping on a validation set, and multi-class softmax regression. Each is a few lines; the loop stays the same.
