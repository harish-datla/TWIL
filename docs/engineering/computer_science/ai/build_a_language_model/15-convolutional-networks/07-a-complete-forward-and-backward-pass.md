# A complete forward and backward pass

The programs below compute the same 3×3 input and 2×2 kernel as the last widget. They assert the hand-computed output (0.1) and gradient (−1.9), then check every weight against a central finite difference. The arrays are small enough that every index is visible, and both programs use double precision, so rounding error stays far below the check's tolerance.
Valid cross-correlation and shared-kernel gradients, complete programs
C++20
Rust 2021

#include <array>
#include <cassert>
#include <cmath>
#include <vector>
struct Result { std::vector<double> output; std::array<double,4> gradient{}; double loss = 0; };
Result evaluate(const std::array<double,9>& input,
                const std::array<double,4>& kernel) {
    Result result;
    // Valid 2x2 cross-correlation on a 3x3 input, one channel, bias zero.
    for (int row = 0; row < 2; ++row) for (int column = 0; column < 2; ++column) {
        double output = 0;
        for (int kernelRow = 0; kernelRow < 2; ++kernelRow)
        for (int kernelColumn = 0; kernelColumn < 2; ++kernelColumn)
            output += input[(row+kernelRow)*3+column+kernelColumn]
                    * kernel[kernelRow*2+kernelColumn];
        result.output.push_back(output);
        const double difference = output - 1.0; // Every target is one in this harness.
        result.loss += 0.5 * difference * difference;
        for (int kernelRow = 0; kernelRow < 2; ++kernelRow)
        for (int kernelColumn = 0; kernelColumn < 2; ++kernelColumn)
            // += collects uses of one shared weight at different output positions.
            result.gradient[kernelRow*2+kernelColumn] += difference
                * input[(row+kernelRow)*3+column+kernelColumn];
    }
    return result;
}
int main() {
    const std::array<double,9> input{1,2,0, 0,1,3, 2,0,1};
    const std::array<double,4> kernel{0.2,-0.1,0.3,0.1};
    const Result result = evaluate(input,kernel);
    assert(std::abs(result.output[0]-0.1) < 1e-12);
    assert(std::abs(result.gradient[0]+1.9) < 1e-12);
    for (int index = 0; index < 4; ++index) {
        auto plus = kernel, minus = kernel;
        const double epsilon = 1e-5;
        plus[index] += epsilon; minus[index] -= epsilon;
        const double numerical = (evaluate(input,plus).loss
                                - evaluate(input,minus).loss)/(2*epsilon);
        // This independently checks the backward pass against the forward loss.
        assert(std::abs(numerical-result.gradient[index]) < 1e-8);
    }
}

The fixed sizes leave out batching, multiple channels, padding, dilation, the bias gradient, and the gradient with respect to the input, which a real layer needs to pass the error back to the layer before it. Adding channels adds one more sum to the forward pass and one more gradient array per kernel. Adding a batch adds a sum over examples. Decide whether the loss is a sum or an average over the batch before you scale the gradient.

To turn this into a small image classifier, run a convolution, apply ReLU, optionally downsample, and repeat. Then average each channel's map to one number and feed those numbers to a dense layer with a softmax. The Neural Networks page derives the gradient for that last layer; the convolution backward pass, plus the input gradient, carries it back into the shared kernels. Production libraries use memory-layout-aware, vectorized or GPU kernels, and a plain scalar loop like this one is a useful reference to test them against.
Check Your Understanding Q1 of 2

Why does the backward kernel use +=?
The gradient is a running optimizer velocity.
Each output has an independent copy of the weight.
The same weight affects several output positions.
0/2 correct
