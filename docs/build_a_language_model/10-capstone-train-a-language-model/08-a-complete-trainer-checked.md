# A complete trainer, checked

These programs are the browser model without the browser. They follow the spirit of llm.c's CPU reference, which trains GPT-2 in about a thousand lines of C with every kernel written out, and fine-tunes the released 124M weights on tinyshakespeare from a validation loss of 5.25 to 4.11 in 40 steps.[18] Each listing holds the one-block GPT, its hand-written backward pass, AdamW with clipping, warmup and cosine decay, random-offset batches with shifted targets, and sampling with temperature. They build their own training text from templated sentences, so they need no data file.

Before training, each program asserts two things. The initial loss must be within 0.05 of ln V. And for every parameter group, the hand-written gradient must agree with a centered finite difference (nudge a weight up and down by a tiny amount and compare the two losses) to a relative error below 10⁻⁴. That second check is what makes a hand-derived backward pass trustworthy. After 400 iterations the program asserts that the validation loss has at least halved, prints it with its perplexity, and samples 120 characters.
A one-block GPT with hand-written gradients, complete programs
C++20
Rust 2021

// A one-block GPT trained on characters, with every gradient written by hand.
// Build: g++ -std=c++20 -O2 -Wall -Wextra -Wpedantic -Werror lm.cpp && ./a.out
#include <algorithm>
#include <cassert>
#include <cctype>
#include <cmath>
#include <cstdint>
#include <cstdio>
#include <numbers>
#include <string>
#include <vector>

struct Rng {  // xorshift32, fixed seed: identical numbers in C++ and Rust
    uint32_t state = 7;
    double next() { state ^= state << 13; state ^= state >> 17; state ^= state << 5; return state / 4294967296.0; }
    double normal() { double u = 0; while (u == 0) u = next(); return std::sqrt(-2 * std::log(u)) * std::cos(2 * std::numbers::pi * next()); }
};

// Hyperparameters of the model and the training run.
constexpr int kBlock = 32, kEmbed = 32, kHeads = 4, kHeadDim = kEmbed / kHeads, kHidden = 4 * kEmbed, kBatch = 8;

struct Model {
    int vocab;
    // Parameters. Every linear map is a plain [in][out] matrix with no bias, like nanoGPT with bias=False.
    std::vector<double> tokenEmbedding, positionEmbedding, gain1, attentionWeight, attentionProjection, gain2, mlpExpand, mlpProjection, gainFinal;
    std::vector<std::vector<double>*> parameters() { return {&tokenEmbedding, &positionEmbedding, &gain1, &attentionWeight, &attentionProjection, &gain2, &mlpExpand, &mlpProjection, &gainFinal}; }
    Model(int vocabSize, Rng& rng) : vocab(vocabSize) {
        auto gaussian = [&](std::size_t size, double std) { std::vector<double> values(size); for (double& value : values) value = rng.normal() * std; return values; };
        tokenEmbedding = gaussian(std::size_t(vocab) * kEmbed, 0.02);   // also the tied output head
        positionEmbedding = gaussian(std::size_t(kBlock) * kEmbed, 0.02);
        gain1.assign(kEmbed, 1.0); gain2.assign(kEmbed, 1.0); gainFinal.assign(kEmbed, 1.0);
        attentionWeight = gaussian(std::size_t(kEmbed) * 3 * kEmbed, 0.02);
        attentionProjection = gaussian(std::size_t(kEmbed) * kEmbed, 0.02 / std::sqrt(2.0));  // GPT-2 residual scaling, 1 layer
        mlpExpand = gaussian(std::size_t(kEmbed) * kHidden, 0.02);
        mlpProjection = gaussian(std::size_t(kHidden) * kEmbed, 0.02 / std::sqrt(2.0));
    }
};

// Gradients, Adam moments: same shapes as the parameters.
struct Shadow {
    std::vector<std::vector<double>> slots;
    explicit Shadow(Model& model) { for (auto* parameter : model.parameters()) slots.emplace_back(parameter->size(), 0.0); }
    void zero() { for (auto& slot : slots) for (double& value : slot) value = 0; }
};

struct NormCache { std::vector<double> output, normalized, inverseStd; };
// LayerNorm without a bias term over rows of width kEmbed.
NormCache layerNormForward(const std::vector<double>& input, const std::vector<double>& gain, int rows) {
    NormCache cache{std::vector<double>(input.size()), std::vector<double>(input.size()), std::vector<double>(rows)};
    for (int row = 0; row < rows; ++row) {
        double mean = 0, variance = 0;
        for (int c = 0; c < kEmbed; ++c) mean += input[row * kEmbed + c];
        mean /= kEmbed;
        for (int c = 0; c < kEmbed; ++c) variance += (input[row * kEmbed + c] - mean) * (input[row * kEmbed + c] - mean);
        cache.inverseStd[row] = 1.0 / std::sqrt(variance / kEmbed + 1e-5);
        for (int c = 0; c < kEmbed; ++c) {
            cache.normalized[row * kEmbed + c] = (input[row * kEmbed + c] - mean) * cache.inverseStd[row];
            cache.output[row * kEmbed + c] = cache.normalized[row * kEmbed + c] * gain[c];
        }
    }
    return cache;
}
// dL/dx = (1/σ) (g⊙dy − mean(g⊙dy) − x̂ · mean(g⊙dy⊙x̂)), accumulated into inputGrad.
void layerNormBackward(const std::vector<double>& outputGrad, const NormCache& cache, const std::vector<double>& gain, std::vector<double>& gainGrad, int rows, std::vector<double>& inputGrad) {
    for (int row = 0; row < rows; ++row) {
        double sumGrad = 0, sumGradNormalized = 0;
        for (int c = 0; c < kEmbed; ++c) {
            double scaled = outputGrad[row * kEmbed + c] * gain[c], normalized = cache.normalized[row * kEmbed + c];
            gainGrad[c] += outputGrad[row * kEmbed + c] * normalized;
            sumGrad += scaled; sumGradNormalized += scaled * normalized;
        }
        for (int c = 0; c < kEmbed; ++c) {
            double scaled = outputGrad[row * kEmbed + c] * gain[c], normalized = cache.normalized[row * kEmbed + c];
            inputGrad[row * kEmbed + c] += cache.inverseStd[row] * (scaled - sumGrad / kEmbed - normalized * sumGradNormalized / kEmbed);
        }
    }
}
// output[rows][outDim] = input[rows][inDim] · weight[inDim][outDim]
std::vector<double> linearForward(const std::vector<double>& input, const std::vector<double>& weight, int rows, int inDim, int outDim) {
    std::vector<double> output(std::size_t(rows) * outDim, 0.0);
    for (int row = 0; row < rows; ++row) for (int i = 0; i < inDim; ++i) {
        double x = input[row * inDim + i];
        for (int o = 0; o < outDim; ++o) output[row * outDim + o] += x * weight[i * outDim + o];
    }
    return output;
}
// weightGrad += inputᵀ · outputGrad; inputGrad += outputGrad · weightᵀ
void linearBackward(const std::vector<double>& outputGrad, const std::vector<double>& input, const std::vector<double>& weight, std::vector<double>& weightGrad, int rows, int inDim, int outDim, std::vector<double>& inputGrad) {
    for (int row = 0; row < rows; ++row) for (int i = 0; i < inDim; ++i) {
        double x = input[row * inDim + i], accumulated = 0;
        for (int o = 0; o < outDim; ++o) { double gy = outputGrad[row * outDim + o]; weightGrad[i * outDim + o] += x * gy; accumulated += gy * weight[i * outDim + o]; }
        inputGrad[row * inDim + i] += accumulated;
    }
}
double gelu(double x) { double u = std::sqrt(2 / std::numbers::pi) * (x + 0.044715 * x * x * x); return 0.5 * x * (1 + std::tanh(u)); }
double geluGrad(double x) { double u = std::sqrt(2 / std::numbers::pi) * (x + 0.044715 * x * x * x), t = std::tanh(u); return 0.5 * (1 + t) + 0.5 * x * (1 - t * t) * std::sqrt(2 / std::numbers::pi) * (1 + 3 * 0.044715 * x * x); }

// Everything the backward pass needs from the forward pass.
struct Activations {
    int batch, length, rows;
    std::vector<int> inputs, targets;
    std::vector<double> qkv, weights, attentionOut, hidden, activated, probabilities;  // the residual sums are not needed again
    NormCache norm1, norm2, normFinal;
    double loss;
};

Activations forward(const Model& model, const std::vector<int>& inputs, const std::vector<int>& targets, int batch, int length) {
    Activations act; act.batch = batch; act.length = length; act.rows = batch * length; act.inputs = inputs; act.targets = targets;
    int rows = act.rows, vocab = model.vocab;
    std::vector<double> residual0(std::size_t(rows) * kEmbed, 0.0);
    for (int b = 0; b < batch; ++b) for (int t = 0; t < length; ++t) for (int c = 0; c < kEmbed; ++c)
        residual0[(b * length + t) * kEmbed + c] = model.tokenEmbedding[inputs[b * length + t] * kEmbed + c] + model.positionEmbedding[t * kEmbed + c];
    act.norm1 = layerNormForward(residual0, model.gain1, rows);
    act.qkv = linearForward(act.norm1.output, model.attentionWeight, rows, kEmbed, 3 * kEmbed);  // queries | keys | values
    act.weights.assign(std::size_t(batch) * kHeads * length * length, 0.0);
    act.attentionOut.assign(std::size_t(rows) * kEmbed, 0.0);
    double scale = 1.0 / std::sqrt(double(kHeadDim));
    for (int b = 0; b < batch; ++b) for (int h = 0; h < kHeads; ++h) for (int t = 0; t < length; ++t) {
        double* row = &act.weights[((b * kHeads + h) * length + t) * length];
        double maximum = -1e300;
        for (int s = 0; s <= t; ++s) {  // causal: only positions s ≤ t are scored
            double score = 0;
            for (int d = 0; d < kHeadDim; ++d) score += act.qkv[(b * length + t) * 3 * kEmbed + h * kHeadDim + d] * act.qkv[(b * length + s) * 3 * kEmbed + kEmbed + h * kHeadDim + d];
            row[s] = score * scale; maximum = std::max(maximum, row[s]);
        }
        double total = 0;
        for (int s = 0; s <= t; ++s) { row[s] = std::exp(row[s] - maximum); total += row[s]; }
        for (int s = 0; s <= t; ++s) {
            row[s] /= total;
            for (int d = 0; d < kHeadDim; ++d) act.attentionOut[(b * length + t) * kEmbed + h * kHeadDim + d] += row[s] * act.qkv[(b * length + s) * 3 * kEmbed + 2 * kEmbed + h * kHeadDim + d];
        }
    }
    std::vector<double> projected = linearForward(act.attentionOut, model.attentionProjection, rows, kEmbed, kEmbed);
    std::vector<double> residual1(residual0.size());
    for (std::size_t i = 0; i < residual1.size(); ++i) residual1[i] = residual0[i] + projected[i];
    act.norm2 = layerNormForward(residual1, model.gain2, rows);
    act.hidden = linearForward(act.norm2.output, model.mlpExpand, rows, kEmbed, kHidden);
    act.activated.resize(act.hidden.size());
    for (std::size_t i = 0; i < act.hidden.size(); ++i) act.activated[i] = gelu(act.hidden[i]);
    std::vector<double> mlpOut = linearForward(act.activated, model.mlpProjection, rows, kHidden, kEmbed);
    std::vector<double> residual2(residual1.size());
    for (std::size_t i = 0; i < residual2.size(); ++i) residual2[i] = residual1[i] + mlpOut[i];
    act.normFinal = layerNormForward(residual2, model.gainFinal, rows);
    // Logits through the tied token embedding, then softmax and mean cross-entropy.
    act.probabilities.assign(std::size_t(rows) * vocab, 0.0);
    act.loss = 0;
    for (int row = 0; row < rows; ++row) {
        double maximum = -1e300;
        for (int v = 0; v < vocab; ++v) {
            double logit = 0;
            for (int c = 0; c < kEmbed; ++c) logit += act.normFinal.output[row * kEmbed + c] * model.tokenEmbedding[v * kEmbed + c];
            act.probabilities[row * vocab + v] = logit; maximum = std::max(maximum, logit);
        }
        double total = 0;
        for (int v = 0; v < vocab; ++v) { act.probabilities[row * vocab + v] = std::exp(act.probabilities[row * vocab + v] - maximum); total += act.probabilities[row * vocab + v]; }
        for (int v = 0; v < vocab; ++v) act.probabilities[row * vocab + v] /= total;
        if (!targets.empty()) act.loss -= std::log(act.probabilities[row * vocab + targets[row]]);
    }
    if (!targets.empty()) act.loss /= rows;
    return act;
}

void backward(const Model& model, const Activations& act, Shadow& grads) {
    int rows = act.rows, batch = act.batch, length = act.length, vocab = model.vocab;
    // Gradient slots in the order of Model::parameters().
    auto& tokenGrad = grads.slots[0]; auto& positionGrad = grads.slots[1]; auto& gain1Grad = grads.slots[2]; auto& attentionGrad = grads.slots[3]; auto& projectionGrad = grads.slots[4];
    auto& gain2Grad = grads.slots[5]; auto& expandGrad = grads.slots[6]; auto& mlpProjectionGrad = grads.slots[7]; auto& gainFinalGrad = grads.slots[8];
    // Softmax + cross-entropy: dL/dlogit = (p − onehot(target)) / rows.
    std::vector<double> logitGrad(act.probabilities);
    for (int row = 0; row < rows; ++row) { for (int v = 0; v < vocab; ++v) logitGrad[row * vocab + v] /= rows; logitGrad[row * vocab + act.targets[row]] -= 1.0 / rows; }
    std::vector<double> normFinalGrad(std::size_t(rows) * kEmbed, 0.0);
    for (int row = 0; row < rows; ++row) for (int v = 0; v < vocab; ++v) {
        double gl = logitGrad[row * vocab + v];
        for (int c = 0; c < kEmbed; ++c) { normFinalGrad[row * kEmbed + c] += gl * model.tokenEmbedding[v * kEmbed + c]; tokenGrad[v * kEmbed + c] += gl * act.normFinal.output[row * kEmbed + c]; }
    }
    std::vector<double> residual2Grad(std::size_t(rows) * kEmbed, 0.0);
    layerNormBackward(normFinalGrad, act.normFinal, model.gainFinal, gainFinalGrad, rows, residual2Grad);
    // residual2 = residual1 + mlpOut: the gradient flows unchanged along the skip and through the MLP.
    std::vector<double> activatedGrad(std::size_t(rows) * kHidden, 0.0);
    linearBackward(residual2Grad, act.activated, model.mlpProjection, mlpProjectionGrad, rows, kHidden, kEmbed, activatedGrad);
    std::vector<double> hiddenGrad(activatedGrad.size());
    for (std::size_t i = 0; i < hiddenGrad.size(); ++i) hiddenGrad[i] = activatedGrad[i] * geluGrad(act.hidden[i]);
    std::vector<double> norm2Grad(std::size_t(rows) * kEmbed, 0.0);
    linearBackward(hiddenGrad, act.norm2.output, model.mlpExpand, expandGrad, rows, kEmbed, kHidden, norm2Grad);
    std::vector<double> residual1Grad(residual2Grad);
    layerNormBackward(norm2Grad, act.norm2, model.gain2, gain2Grad, rows, residual1Grad);
    std::vector<double> attentionOutGrad(std::size_t(rows) * kEmbed, 0.0);
    linearBackward(residual1Grad, act.attentionOut, model.attentionProjection, projectionGrad, rows, kEmbed, kEmbed, attentionOutGrad);
    std::vector<double> qkvGrad(std::size_t(rows) * 3 * kEmbed, 0.0);
    double scale = 1.0 / std::sqrt(double(kHeadDim));
    std::vector<double> weightGrad(length);
    for (int b = 0; b < batch; ++b) for (int h = 0; h < kHeads; ++h) for (int t = 0; t < length; ++t) {
        const double* row = &act.weights[((b * kHeads + h) * length + t) * length];
        double dotted = 0;
        for (int s = 0; s <= t; ++s) {  // through the weighted sum of values
            double accumulated = 0;
            for (int d = 0; d < kHeadDim; ++d) {
                double go = attentionOutGrad[(b * length + t) * kEmbed + h * kHeadDim + d];
                accumulated += go * act.qkv[(b * length + s) * 3 * kEmbed + 2 * kEmbed + h * kHeadDim + d];
                qkvGrad[(b * length + s) * 3 * kEmbed + 2 * kEmbed + h * kHeadDim + d] += row[s] * go;
            }
            weightGrad[s] = accumulated; dotted += row[s] * accumulated;
        }
        for (int s = 0; s <= t; ++s) {  // softmax backward, then the scaled dot product
            double scoreGrad = row[s] * (weightGrad[s] - dotted) * scale;
            for (int d = 0; d < kHeadDim; ++d) {
                qkvGrad[(b * length + t) * 3 * kEmbed + h * kHeadDim + d] += scoreGrad * act.qkv[(b * length + s) * 3 * kEmbed + kEmbed + h * kHeadDim + d];
                qkvGrad[(b * length + s) * 3 * kEmbed + kEmbed + h * kHeadDim + d] += scoreGrad * act.qkv[(b * length + t) * 3 * kEmbed + h * kHeadDim + d];
            }
        }
    }
    std::vector<double> norm1Grad(std::size_t(rows) * kEmbed, 0.0);
    linearBackward(qkvGrad, act.norm1.output, model.attentionWeight, attentionGrad, rows, kEmbed, 3 * kEmbed, norm1Grad);
    std::vector<double> residual0Grad(residual1Grad);
    layerNormBackward(norm1Grad, act.norm1, model.gain1, gain1Grad, rows, residual0Grad);
    for (int b = 0; b < batch; ++b) for (int t = 0; t < length; ++t) for (int c = 0; c < kEmbed; ++c) {
        tokenGrad[act.inputs[b * length + t] * kEmbed + c] += residual0Grad[(b * length + t) * kEmbed + c];
        positionGrad[t * kEmbed + c] += residual0Grad[(b * length + t) * kEmbed + c];
    }
}

// AdamW: decoupled weight decay on matrices only, bias-corrected moments, after clipping the global gradient norm to 1.
void adamStep(Model& model, Shadow& grads, Shadow& moment, Shadow& second, int step, double rate) {
    double squared = 0;
    for (auto& slot : grads.slots) for (double value : slot) squared += value * value;
    double clip = std::min(1.0, 1.0 / (std::sqrt(squared) + 1e-6));
    const double beta1 = 0.9, beta2 = 0.99, epsilon = 1e-8, weightDecay = 0.1;
    auto parameters = model.parameters();
    for (std::size_t p = 0; p < parameters.size(); ++p) {
        bool isMatrix = parameters[p]->size() > std::size_t(kEmbed);  // gains are vectors: no decay
        for (std::size_t i = 0; i < parameters[p]->size(); ++i) {
            double gradient = grads.slots[p][i] * clip;
            moment.slots[p][i] = beta1 * moment.slots[p][i] + (1 - beta1) * gradient;
            second.slots[p][i] = beta2 * second.slots[p][i] + (1 - beta2) * gradient * gradient;
            double correctedMoment = moment.slots[p][i] / (1 - std::pow(beta1, step)), correctedSecond = second.slots[p][i] / (1 - std::pow(beta2, step));
            (*parameters[p])[i] -= rate * (correctedMoment / (std::sqrt(correctedSecond) + epsilon) + (isMatrix ? weightDecay : 0.0) * (*parameters[p])[i]);
        }
    }
}

int main() {
    // A synthetic corpus of templated sentences, so the program needs no data file.
    const std::vector<std::string> subjects{"the cat", "a dog", "the old owl", "my horse", "one fox"}, verbs{"sat on", "ran past", "looked at", "slept under"}, objects{"the mat", "a red barn", "the gate", "our wall"};
    Rng rng;
    std::string text;
    while (text.size() < 24000) {
        // Separate declarations fix the draw order: the operands of + are unsequenced in C++, so three rng.next() calls in one expression run in a compiler-chosen order.
        const std::size_t subject = std::size_t(rng.next() * subjects.size()), verb = std::size_t(rng.next() * verbs.size()), object = std::size_t(rng.next() * objects.size());
        std::string sentence = subjects[subject] + " " + verbs[verb] + " " + objects[object] + ".";
        sentence[0] = char(std::toupper(sentence[0]));
        text += sentence + (rng.next() < 0.3 ? "\n" : " ");
    }
    std::vector<int> charToId(256, -1); std::string vocabulary;
    for (unsigned char ch : text) if (charToId[ch] < 0) { charToId[ch] = int(vocabulary.size()); vocabulary.push_back(char(ch)); }
    int vocab = int(vocabulary.size());
    std::vector<int> data; for (unsigned char ch : text) data.push_back(charToId[ch]);
    std::size_t split = data.size() * 9 / 10;  // 90% train, 10% validation, in document order
    std::printf("%zu characters, vocabulary %d, ln V = %.3f\n", text.size(), vocab, std::log(double(vocab)));

    Model model(vocab, rng);
    Shadow grads(model), moment(model), second(model);
    auto sampleBatch = [&](std::size_t begin, std::size_t end, std::vector<int>& inputs, std::vector<int>& targets) {
        inputs.assign(std::size_t(kBatch) * kBlock, 0); targets.assign(std::size_t(kBatch) * kBlock, 0);
        for (int b = 0; b < kBatch; ++b) {
            std::size_t offset = begin + std::size_t(rng.next() * double(end - begin - kBlock - 1));  // random window
            for (int t = 0; t < kBlock; ++t) { inputs[b * kBlock + t] = data[offset + t]; targets[b * kBlock + t] = data[offset + t + 1]; }  // targets shifted by one
        }
    };
    std::vector<int> inputs, targets;

    // Check 1: at initialization the loss is close to ln V, because the logits are near zero.
    sampleBatch(0, split, inputs, targets);
    Activations act = forward(model, inputs, targets, kBatch, kBlock);
    assert(std::fabs(act.loss - std::log(double(vocab))) < 0.05);

    // Check 2: the hand-written backward pass agrees with centered finite differences on every parameter group.
    grads.zero(); backward(model, act, grads);
    auto parameters = model.parameters();
    double worstRelativeError = 0;
    for (std::size_t p = 0; p < parameters.size(); ++p) for (std::size_t i : {std::size_t(0), parameters[p]->size() / 2, parameters[p]->size() - 1}) {
        double original = (*parameters[p])[i], h = 1e-5;
        (*parameters[p])[i] = original + h; double lossPlus = forward(model, inputs, targets, kBatch, kBlock).loss;
        (*parameters[p])[i] = original - h; double lossMinus = forward(model, inputs, targets, kBatch, kBlock).loss;
        (*parameters[p])[i] = original;
        double numeric = (lossPlus - lossMinus) / (2 * h), analytic = grads.slots[p][i];
        worstRelativeError = std::max(worstRelativeError, std::fabs(numeric - analytic) / std::max(1e-6, std::fabs(numeric) + std::fabs(analytic)));
    }
    std::printf("gradient check: worst relative error %.2e\n", worstRelativeError);
    assert(worstRelativeError < 1e-4);  // double precision; the finite difference itself is accurate to roughly 1e-6

    // Train: warmup then cosine decay, as in nanoGPT's get_lr.
    const int iterations = 400, warmup = 40; const double maxRate = 3e-3, minRate = 3e-4;
    double initialLoss = act.loss, finalValidation = 0;
    for (int step = 1; step <= iterations; ++step) {
        double rate = step <= warmup ? maxRate * step / warmup : minRate + 0.5 * (1 + std::cos(std::numbers::pi * double(step - warmup) / (iterations - warmup))) * (maxRate - minRate);
        sampleBatch(0, split, inputs, targets);
        grads.zero();
        act = forward(model, inputs, targets, kBatch, kBlock);
        backward(model, act, grads);
        adamStep(model, grads, moment, second, step, rate);
        if (step % 100 == 0 || step == iterations) {
            double validation = 0;
            for (int k = 0; k < 5; ++k) { sampleBatch(split, data.size(), inputs, targets); validation += forward(model, inputs, targets, kBatch, kBlock).loss; }
            finalValidation = validation / 5;
            std::printf("step %4d  lr %.2e  train %.3f  val %.3f  val perplexity %.2f\n", step, rate, act.loss, finalValidation, std::exp(finalValidation));
        }
    }
    assert(finalValidation < initialLoss / 2);  // the model learned something: validation loss at least halved

    // Sample 120 characters with temperature 0.8 from a prompt, cropping the context to the block size.
    std::vector<int> context; for (char ch : std::string("The cat ")) context.push_back(charToId[(unsigned char)ch]);
    std::string sample;
    for (int n = 0; n < 120; ++n) {
        std::vector<int> window(context.end() - std::min<std::size_t>(context.size(), kBlock), context.end());
        Activations out = forward(model, window, {}, 1, int(window.size()));
        std::vector<double> logits(vocab);
        double maximum = -1e300;
        for (int v = 0; v < vocab; ++v) { logits[v] = std::log(out.probabilities[(window.size() - 1) * vocab + v]) / 0.8; maximum = std::max(maximum, logits[v]); }
        double total = 0; for (double& logit : logits) { logit = std::exp(logit - maximum); total += logit; }
        double u = rng.next() * total; int chosen = 0;
        while (chosen < vocab - 1 && (u -= logits[chosen]) > 0) ++chosen;
        context.push_back(chosen); sample.push_back(vocabulary[chosen]);
    }
    std::printf("sample: %s\n", sample.c_str());
    return 0;
}

What's intentionally missing

Dropout, bias vectors, more than one block, mixed precision, gradient accumulation, checkpoint saving, a real tokenizer, data loading from disk, and any kernel faster than three nested loops. Generation also skips the key-value cache, which stores each earlier position's keys and values so that a new character costs one position's work; without it, every new character reruns the whole window through the model. The forward pass, loss, gradients, optimizer and schedule are the ones nanoGPT runs, except that GELU uses its tanh approximation where nanoGPT uses the exact form.
