# A checked integer dot product

The two programs below, in C++ and Rust with the same logic, store four values as unsigned 8-bit codes, compute the dot product in integers, and convert to a real number at the end. They use the widget's vectors, so the float answer is 1.56. The assertions check that the integer result equals the dot product of the read-back vectors, that an exact 0 stays exact, and that the result lands within 0.04 of 1.56. They also check that huge inputs clip to codes 0 and 255 and that exact halves round away from zero. Four terms keep the accumulator at or below 4 × 65,025 = 260,100. The codes sit in 32-bit integers for readability rather than packed bytes.
Affine coding and a widened integer dot product, complete programs
C++20
Rust 2021

#include <algorithm>
#include <array>
#include <cassert>
#include <cmath>
#include <cstdint>
struct Quantized { std::array<std::int32_t,4> codes; double scale; std::int32_t zero; };
Quantized quantize(const std::array<double,4>& values, double low, double high) {
    assert(std::isfinite(low) && std::isfinite(high));
    assert(low <= 0 && high >= 0 && high > low);
    const double scale = (high-low)/255.0;
    assert(std::isfinite(scale) && scale > 0);
    // Include zero exactly; its code is rounded with the same tie convention.
    const std::int32_t zero = static_cast<std::int32_t>(std::round(-low/scale));
    Quantized result{{},scale,zero};
    for (int index = 0; index < 4; ++index) {
        assert(std::isfinite(values[index]));
        // Clip while still floating point so huge finite inputs cannot overflow an integer cast.
        const double code = std::clamp(std::round(values[index]/scale)+zero,0.0,255.0);
        result.codes[index] = static_cast<std::int32_t>(code);
    }
    return result;
}
double dot(const Quantized& left, const Quantized& right) {
    std::int32_t accumulator = 0;
    for (int index = 0; index < 4; ++index)
        // Codes are already widened; centering removes offsets before multiplication.
        accumulator += (left.codes[index]-left.zero)*(right.codes[index]-right.zero);
    return left.scale*right.scale*accumulator; // Return a real value for this reference.
}
int main() {
    const auto left = quantize({-1,0.3,2,0.1},-2,3);
    const auto right = quantize({0.5,-0.2,1,1.2},-1,2);
    double reconstructed = 0;
    for (int index = 0; index < 4; ++index)
        reconstructed += left.scale*(left.codes[index]-left.zero)
                       * right.scale*(right.codes[index]-right.zero);
    assert(std::abs(dot(left,right)-reconstructed) < 1e-12);
    const auto zeros = quantize({0,0,0,0},-2,3);
    assert(dot(zeros,right) == 0); // Exact real zero survives the affine representation.
    assert(std::abs(dot(left,right)-1.56) < 0.04); // Float dot is 1.56 for this input.
    const auto extremes = quantize({1e100,-1e100,0,1},-2,3);
    assert(extremes.codes[0] == 255 && extremes.codes[1] == 0); // Clipping precedes conversion.
    const auto ties = quantize({0.5,-0.5,0,0},-127.5,127.5);
    assert(ties.codes[0]-ties.zero == 1 && ties.codes[1]-ties.zero == -1);
}

What's intentionally missing

This is a reference calculation, written to be easy to check. It doesn't pack 4-bit codes two to a byte, use vector instructions, fuse the bias and activation into the multiply, or convert the result back into codes for the next layer. A deployment needs each of those to match its model graph and its hardware backend. The browser widgets likewise show representations and results; none of them is an int8 speed test.

Keep representative inputs and the unquantized model's outputs. Then compare in order: how well each tensor survives the round trip, how far each layer's outputs move, how the real task scores on held-out data, and finally how fast the actual deployment path runs at realistic batch sizes and sequence lengths. Each step answers a different question, and passing one doesn't imply passing the next.
Check Your Understanding Q1 of 2

Why does the corrected integer result equal the dequantized-vector dot?
Because quantization is lossless.
Because every zero point is zero.
Both expand to the same scale product and centered integer sum.
0/2 correct
