# Skip-gram training, checked

These complete programs run the same experiment as the first widget, in both languages. They generate a two-topic corpus, build a noise table from counts raised to 0.75, train skip-gram vectors with negative sampling for a few epochs, and then assert that the geometry came out right: "cat" is closer by cosine to "dog" than to "car", "car" is closer to "truck" than to "sheep", and the training loss fell. The random numbers come from a fixed linear congruential generator, a one-line formula that produces the same sequence on every run, so both languages see identical data.
Skip-gram with negative sampling, complete programs
C++20
Rust 2021

#include <cassert>
#include <cmath>
#include <cstddef>
#include <cstdint>
#include <string>
#include <vector>
struct Rng { uint64_t state = 12345; double next() { state = state * 6364136223846793005ULL + 1442695040888963407ULL; return double(state >> 11) / 9007199254740992.0; } };
const std::vector<std::string> vocabulary{"the", "in", "on", "cat", "dog", "cow", "sheep", "eats", "sleeps", "barn", "field", "car", "truck", "bus", "bike", "drives", "stops", "road", "garage"};
int id(const std::string& word) { for (std::size_t index = 0; index < vocabulary.size(); ++index) if (vocabulary[index] == word) return int(index); return -1; }
// Templated sentences: "the <subject> <verb> in|on the <place>", half farm, half traffic.
std::vector<int> corpus(Rng& rng, int sentences) {
    const std::vector<std::vector<std::string>> farm{{"cat", "dog", "cow", "sheep"}, {"eats", "sleeps"}, {"barn", "field"}};
    const std::vector<std::vector<std::string>> traffic{{"car", "truck", "bus", "bike"}, {"drives", "stops"}, {"road", "garage"}};
    std::vector<int> tokens;
    for (int sentence = 0; sentence < sentences; ++sentence) {
        const auto& topic = rng.next() < 0.5 ? farm : traffic;
        auto pick = [&](const std::vector<std::string>& options) { return id(options[std::size_t(rng.next() * options.size())]); };
        for (int token : {id("the"), pick(topic[0]), pick(topic[1]), rng.next() < 0.5 ? id("in") : id("on"), id("the"), pick(topic[2])}) tokens.push_back(token);
    }
    return tokens;
}
double sigmoid(double value) { return 1 / (1 + std::exp(-value)); }
double cosine(const std::vector<double>& left, const std::vector<double>& right) {
    double product = 0, leftNorm = 0, rightNorm = 0;
    for (std::size_t index = 0; index < left.size(); ++index) { product += left[index] * right[index]; leftNorm += left[index] * left[index]; rightNorm += right[index] * right[index]; }
    return product / std::sqrt(leftNorm * rightNorm);
}
int main() {
    Rng rng;
    const std::vector<int> tokens = corpus(rng, 600);
    const std::size_t dimension = 8, words = vocabulary.size();
    std::vector<double> counts(words, 0);
    for (int token : tokens) counts[token] += 1;
    std::vector<double> noiseWeight(words); double noiseTotal = 0;
    for (std::size_t word = 0; word < words; ++word) { noiseWeight[word] = std::pow(counts[word], 0.75); noiseTotal += noiseWeight[word]; } // word2vec's noise distribution.
    auto sampleNoise = [&]() { double threshold = rng.next() * noiseTotal; for (std::size_t word = 0; word < words; ++word) { threshold -= noiseWeight[word]; if (threshold <= 0) return int(word); } return int(words) - 1; };
    std::vector<std::vector<double>> center(words, std::vector<double>(dimension)), context(words, std::vector<double>(dimension, 0));
    for (auto& vector : center) for (double& value : vector) value = (rng.next() - 0.5) / dimension; // Small random centers, zero contexts, as in word2vec.
    const int window = 2, negatives = 5; const double rate = 0.05;
    double firstLoss = 0, lastLoss = 0;
    for (int epoch = 0; epoch < 8; ++epoch) {
        double loss = 0; int pairs = 0;
        for (int position = 0; position < int(tokens.size()); ++position)
            for (int offset = -window; offset <= window; ++offset) {
                if (offset == 0 || position + offset < 0 || position + offset >= int(tokens.size())) continue;
                const int centerWord = tokens[position];
                std::vector<double> gradientCenter(dimension, 0);
                for (int draw = 0; draw <= negatives; ++draw) {
                    const int target = draw == 0 ? tokens[position + offset] : sampleNoise(); // One observed pair, then K noise words.
                    const double label = draw == 0 ? 1 : 0;
                    double score = 0;
                    for (std::size_t coordinate = 0; coordinate < dimension; ++coordinate) score += center[centerWord][coordinate] * context[target][coordinate];
                    const double probability = sigmoid(score), error = probability - label; // ∂loss/∂score = σ − label.
                    loss -= label ? std::log(probability) : std::log(1 - probability);
                    for (std::size_t coordinate = 0; coordinate < dimension; ++coordinate) {
                        gradientCenter[coordinate] += error * context[target][coordinate];
                        context[target][coordinate] -= rate * error * center[centerWord][coordinate];
                    }
                }
                for (std::size_t coordinate = 0; coordinate < dimension; ++coordinate) center[centerWord][coordinate] -= rate * gradientCenter[coordinate];
                ++pairs;
            }
        if (epoch == 0) firstLoss = loss / pairs;
        lastLoss = loss / pairs;
    }
    assert(lastLoss < firstLoss);
    assert(cosine(center[id("cat")], center[id("dog")]) > cosine(center[id("cat")], center[id("car")]));     // Same topic beats other topic.
    assert(cosine(center[id("car")], center[id("truck")]) > cosine(center[id("car")], center[id("sheep")]));
}

What's intentionally missing

Subsampling of frequent words, the randomly shrunk window, the learning-rate decay, the precomputed sigmoid table and the unigram table of 10⁸ slots that make the reference implementation fast, multithreaded training with unsynchronized updates, phrase detection, and any evaluation beyond three cosine comparisons. The update rule is the one word2vec runs.
