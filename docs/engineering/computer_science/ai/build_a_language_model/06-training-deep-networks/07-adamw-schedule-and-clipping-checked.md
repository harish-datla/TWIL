# AdamW, schedule and clipping, checked

These complete programs implement AdamW with bias correction, gradient clipping by the combined norm, and the warmup-plus-cosine schedule, then check the claims this page made about them. The first Adam step moves every parameter by exactly η whatever the gradient's scale. Clipping keeps the gradient's direction and caps its length. The schedule hits its documented values at the boundaries. And AdamW drives the loss on the condition-50 bowl from section 3 below 10⁻⁸. Both listings use parallel names so you can read them side by side.
AdamW, clipping and schedule, complete programs
C++20
Rust 2021

#include <cassert>
#include <cmath>
#include <cstddef>
#include <numbers>
#include <vector>
using Vector = std::vector<double>;
struct AdamW {
    double beta1, beta2, epsilon, weightDecay;
    Vector firstMoment, secondMoment;
    int step = 0;
    AdamW(std::size_t size, double decay)
        : beta1(0.9), beta2(0.999), epsilon(1e-8), weightDecay(decay), firstMoment(size, 0), secondMoment(size, 0) {}
    void update(Vector& parameters, const Vector& gradient, double scheduledRate) {
        ++step;
        const double correction1 = 1 - std::pow(beta1, step), correction2 = 1 - std::pow(beta2, step);
        for (std::size_t index = 0; index < parameters.size(); ++index) {
            parameters[index] -= scheduledRate * weightDecay * parameters[index]; // Decoupled decay, applied to θ directly.
            firstMoment[index] = beta1 * firstMoment[index] + (1 - beta1) * gradient[index];
            secondMoment[index] = beta2 * secondMoment[index] + (1 - beta2) * gradient[index] * gradient[index];
            const double correctedFirst = firstMoment[index] / correction1, correctedSecond = secondMoment[index] / correction2;
            parameters[index] -= scheduledRate * correctedFirst / (std::sqrt(correctedSecond) + epsilon);
        }
    }
};
// Scale the whole gradient so its global ℓ2 norm is at most maxNorm; returns the norm before clipping.
double clipGlobalNorm(Vector& gradient, double maxNorm) {
    double sumSquares = 0;
    for (double value : gradient) sumSquares += value * value;
    const double norm = std::sqrt(sumSquares);
    if (norm > maxNorm) for (double& value : gradient) value *= maxNorm / norm;
    return norm;
}
// Linear warmup to peak, cosine decay to floor, constant floor afterwards.
double warmupCosine(int iteration, int warmup, int total, double peak, double floor) {
    if (iteration < warmup) return peak * (iteration + 1) / (warmup + 1);
    if (iteration > total) return floor;
    const double ratio = double(iteration - warmup) / (total - warmup);
    return floor + 0.5 * (1 + std::cos(std::numbers::pi * ratio)) * (peak - floor);
}
int main() {
    Vector parameters{0.5, -0.5};
    AdamW firstStep(2, 0.0);
    firstStep.update(parameters, Vector{1000.0, -0.001}, 0.01);
    assert(std::abs(parameters[0] - 0.49) < 1e-6 && std::abs(parameters[1] + 0.49) < 1e-6); // |step| = η per coordinate, whatever the gradient scale (up to ε).
    Vector gradient{3.0, 4.0};
    const double before = clipGlobalNorm(gradient, 1.0);
    assert(std::abs(before - 5.0) < 1e-12 && std::abs(gradient[0] - 0.6) < 1e-12 && std::abs(gradient[1] - 0.8) < 1e-12); // Direction kept, norm capped at 1.
    assert(std::abs(warmupCosine(0, 2000, 20000, 6e-4, 6e-5) - 6e-4 / 2001) < 1e-15);
    assert(std::abs(warmupCosine(2000, 2000, 20000, 6e-4, 6e-5) - 6e-4) < 1e-15);   // Peak at the end of warmup.
    assert(std::abs(warmupCosine(20000, 2000, 20000, 6e-4, 6e-5) - 6e-5) < 1e-15);  // Floor at the end.
    Vector point{-2.5, 2.5};
    AdamW optimizer(2, 0.0);
    for (int iteration = 0; iteration < 2000; ++iteration) {
        Vector bowlGradient{point[0], 50.0 * point[1]}; // ∇ of ½(x² + 50 y²): condition number 50.
        clipGlobalNorm(bowlGradient, 100.0);
        optimizer.update(point, bowlGradient, warmupCosine(iteration, 50, 2000, 0.1, 0.001));
    }
    const double loss = 0.5 * (point[0] * point[0] + 50.0 * point[1] * point[1]);
    assert(loss < 1e-8);
}

What's intentionally missing

Parameter groups that exempt biases and normalization gains from weight decay, the fused and multi-tensor kernels that apply the update to millions of parameters at once, mixed-precision master weights with a loss scaler, gradient accumulation across micro-batches, distributed all-reduce of gradients, and checkpointing of the optimizer state. nanoGPT's training script wires all of these around the same arithmetic.[9]
