# A deterministic DDIM step in two languages

The programs below implement one deterministic DDIM step, from ᾱ = 0.4 to 0.7, on the section 01 example. Their checks feed in the exact noise, −0.8, so the results are known in advance: the clean estimate must be 2, a step all the way to ᾱ = 1 must land on 2, and the velocity conversion must recover 2. There's no trained model in the code.
An asserted deterministic reverse step, complete programs
C++20
Rust 2021

#include <cassert>
#include <cmath>
struct Step { double clean; double next; };
Step ddimStep(double noisy, double predictedNoise,
              double alphaBar, double nextAlphaBar) {
    assert(alphaBar > 0 && alphaBar < 1);
    assert(nextAlphaBar >= alphaBar && nextAlphaBar <= 1);
    // eta=0: no fresh random draw enters this reverse update.
    // Invert the known corruption using this level's prediction.
    const double clean = (noisy-std::sqrt(1-alphaBar)*predictedNoise)
                       / std::sqrt(alphaBar);
    // Reuse that prediction as the deterministic direction at the next level.
    const double next = std::sqrt(nextAlphaBar)*clean
                      + std::sqrt(1-nextAlphaBar)*predictedNoise;
    return {clean,next};
}
int main() {
    const double clean = 2.0, noise = -0.8;
    const double alphaBar = 0.4, nextAlphaBar = 0.7;
    const double noisy = std::sqrt(alphaBar)*clean+std::sqrt(1-alphaBar)*noise;
    const Step result = ddimStep(noisy,noise,alphaBar,nextAlphaBar);
    assert(std::abs(result.clean-clean) < 1e-12);
    const double expected = std::sqrt(nextAlphaBar)*clean
                          + std::sqrt(1-nextAlphaBar)*noise;
    assert(std::abs(result.next-expected) < 1e-12);
    const Step final = ddimStep(noisy,noise,alphaBar,1.0);
    assert(std::abs(final.next-clean) < 1e-12); // Final noise coefficient is zero.
    const double velocity = std::sqrt(alphaBar)*noise-std::sqrt(1-alphaBar)*clean;
    const double fromVelocity = std::sqrt(alphaBar)*noisy-std::sqrt(1-alphaBar)*velocity;
    assert(std::abs(fromVelocity-clean) < 1e-12);
}

A full sampler also needs a model call at every level, the same time input the model was trained with, the model's prediction type, and the same data or latent scaling used in training. A stochastic sampler needs its own noise-variance formula and independent random draws; borrowing the variance formula from a different sampler changes the method.

The program only accepts current noise levels with 0 < ᾱ < 1. At ᾱ = 0 the clean estimate would divide by zero, and ᾱ = 1 means the sample is already clean. The widget starts at a small positive ᾱ and treats the final target level as ᾱ = 1. Code that evaluates one of these endpoints needs to define what happens there instead of dividing by zero.
Check Your Understanding Q1 of 2

Can a sampler use a v prediction directly as an ε prediction?
Yes, because both are standard normal draws.
Only if the number of sampling steps is even.
No; it must convert using the current coefficients.
0/2 correct
