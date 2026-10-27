# The metrics, checked

These complete programs implement the confusion-matrix metrics, the ROC AUC computed two ways, expected calibration error, the pass@k estimator, and perplexity from a list of token log-likelihoods. Each one is checked against a value worked out by hand: the two AUC methods agree on a ranked example, pass@k matches a direct count of combinations, and ECE matches a two-bin calculation. Both listings use parallel names so you can read them side by side.
Evaluation metrics, complete programs
C++20
Rust 2021

#include <algorithm>
#include <cassert>
#include <cmath>
#include <cstddef>
#include <vector>
struct Scored { double score; int label; };
struct Metrics { double precision, recall, f1, accuracy; };
Metrics thresholdMetrics(const std::vector<Scored>& data, double threshold) {
    int truePositive = 0, falsePositive = 0, falseNegative = 0, trueNegative = 0;
    for (const Scored& example : data) {
        const bool predicted = example.score >= threshold;
        if (predicted && example.label) ++truePositive; else if (predicted) ++falsePositive;
        else if (example.label) ++falseNegative; else ++trueNegative;
    }
    const double precision = double(truePositive) / (truePositive + falsePositive), recall = double(truePositive) / (truePositive + falseNegative);
    return {precision, recall, 2 * precision * recall / (precision + recall), double(truePositive + trueNegative) / data.size()};
}
// AUC as the fraction of (positive, negative) pairs the scores rank correctly; ties count one half.
double rankAuc(const std::vector<Scored>& data) {
    double concordant = 0; int pairs = 0;
    for (const Scored& positive : data) if (positive.label)
        for (const Scored& negative : data) if (!negative.label) {
            ++pairs; concordant += positive.score > negative.score ? 1.0 : positive.score == negative.score ? 0.5 : 0.0;
        }
    return concordant / pairs;
}
// AUC by the trapezoid rule under the ROC curve traced by sweeping the threshold downward.
double trapezoidAuc(std::vector<Scored> data) {
    std::sort(data.begin(), data.end(), [](const Scored& left, const Scored& right) { return left.score > right.score; });
    int positives = 0; for (const Scored& example : data) positives += example.label;
    const int negatives = int(data.size()) - positives;
    double area = 0, previousFpr = 0, previousTpr = 0; int truePositive = 0, falsePositive = 0;
    for (const Scored& example : data) {
        if (example.label) ++truePositive; else ++falsePositive;
        const double fpr = double(falsePositive) / negatives, tpr = double(truePositive) / positives;
        area += (fpr - previousFpr) * (tpr + previousTpr) / 2; previousFpr = fpr; previousTpr = tpr;
    }
    return area;
}
// Expected calibration error over equal-width confidence bins.
double expectedCalibrationError(const std::vector<double>& confidences, const std::vector<int>& correct, int bins) {
    std::vector<double> sumConfidence(bins, 0), sumCorrect(bins, 0); std::vector<int> count(bins, 0);
    for (std::size_t index = 0; index < confidences.size(); ++index) {
        const int bin = std::min(bins - 1, int(confidences[index] * bins));
        ++count[bin]; sumConfidence[bin] += confidences[index]; sumCorrect[bin] += correct[index];
    }
    double error = 0;
    for (int bin = 0; bin < bins; ++bin) if (count[bin])
        error += double(count[bin]) / confidences.size() * std::abs(sumCorrect[bin] / count[bin] - sumConfidence[bin] / count[bin]);
    return error;
}
// HumanEval's unbiased pass@k from n samples of which c passed: 1 − C(n−c, k) / C(n, k).
double passAtK(int samples, int passed, int k) {
    if (samples - passed < k) return 1.0;
    double product = 1;
    for (int index = samples - passed + 1; index <= samples; ++index) product *= 1.0 - double(k) / index;
    return 1.0 - product;
}
double perplexity(const std::vector<double>& logProbabilities) {
    double total = 0;
    for (double value : logProbabilities) total += value;
    return std::exp(-total / logProbabilities.size()); // exp of the mean negative log-likelihood.
}
int main() {
    const std::vector<Scored> data{{0.9, 1}, {0.8, 1}, {0.7, 0}, {0.6, 1}, {0.4, 0}, {0.3, 0}, {0.2, 1}, {0.1, 0}};
    const Metrics metrics = thresholdMetrics(data, 0.5); // Predicted positive: 0.9, 0.8, 0.7, 0.6 → TP 3, FP 1, FN 1, TN 3.
    assert(std::abs(metrics.precision - 0.75) < 1e-12 && std::abs(metrics.recall - 0.75) < 1e-12);
    assert(std::abs(metrics.f1 - 0.75) < 1e-12 && std::abs(metrics.accuracy - 0.75) < 1e-12);
    assert(std::abs(rankAuc(data) - 12.0 / 16.0) < 1e-12);                    // 12 of the 16 positive–negative pairs are ranked correctly.
    assert(std::abs(trapezoidAuc(data) - rankAuc(data)) < 1e-12);          // Both methods agree.
    const std::vector<double> confidences{0.95, 0.95, 0.95, 0.95, 0.55, 0.55};
    const std::vector<int> correct{1, 1, 1, 0, 1, 0};             // 0.95 bin: 75% right; 0.55 bin: 50% right.
    assert(std::abs(expectedCalibrationError(confidences, correct, 10) - (4.0 / 6 * 0.20 + 2.0 / 6 * 0.05)) < 1e-12);
    assert(std::abs(passAtK(10, 3, 1) - 0.3) < 1e-12);                      // One sample: the plain pass rate.
    assert(std::abs(passAtK(10, 3, 5) - (1.0 - 21.0 / 252.0)) < 1e-12);     // C(7,5) = 21 all-failing draws of C(10,5) = 252.
    assert(std::abs(perplexity({std::log(0.5), std::log(0.5), std::log(0.5)}) - 2.0) < 1e-12); // Coin-flip tokens: perplexity 2.
}

What's intentionally missing

Multi-class averaging (macro, micro, weighted), confidence intervals around each metric, bootstrap resampling for paired model comparisons, the striding logic for perplexity over long texts, and the sandboxed execution that makes pass@k safe to compute on generated code.
