# A checked matrix kernel

These complete programs build a small matrix type stored row by row, multiply in the row-friendly loop order, and check three claims from this page: the fast order gives the same answer as the textbook order, a rotation keeps a vector's length, and a column times a row gives a matrix where every row is a multiple of the same vector (rank one). Both listings use the same names so you can read them side by side.
Row-major matrices, complete programs
C++20
Rust 2021

#include <cassert>
#include <cmath>
#include <cstddef>
#include <vector>
struct Matrix {
    std::size_t rows, columns;
    std::vector<double> data; // Row-major: element (row, column) sits at row*columns + column.
    Matrix(std::size_t rowCount, std::size_t columnCount)
        : rows(rowCount), columns(columnCount), data(rowCount * columnCount, 0.0) {}
    double& at(std::size_t row, std::size_t column) { return data[row * columns + column]; }
    double at(std::size_t row, std::size_t column) const { return data[row * columns + column]; }
};
// Row-friendly order: the inner loop walks along a row of right and a row of product,
// so consecutive reads and writes are adjacent in memory.
Matrix multiply(const Matrix& left, const Matrix& right) {
    assert(left.columns == right.rows); // Inner sizes must agree.
    Matrix product(left.rows, right.columns);
    for (std::size_t row = 0; row < left.rows; ++row)
        for (std::size_t inner = 0; inner < left.columns; ++inner) {
            const double leftValue = left.at(row, inner); // Constant across the inner loop.
            for (std::size_t column = 0; column < right.columns; ++column)
                product.at(row, column) += leftValue * right.at(inner, column);
        }
    return product;
}
Matrix multiplyTextbook(const Matrix& left, const Matrix& right) {
    Matrix product(left.rows, right.columns);
    for (std::size_t row = 0; row < left.rows; ++row)
        for (std::size_t column = 0; column < right.columns; ++column)
            for (std::size_t inner = 0; inner < left.columns; ++inner)
                product.at(row, column) += left.at(row, inner) * right.at(inner, column);
    return product;
}
double norm(const Matrix& vector) {
    double sum = 0;
    for (double value : vector.data) sum += value * value;
    return std::sqrt(sum);
}
int main() {
    Matrix left(3, 4), right(4, 5);
    for (std::size_t index = 0; index < left.data.size(); ++index) left.data[index] = double(index % 7) - 3;
    for (std::size_t index = 0; index < right.data.size(); ++index) right.data[index] = double(index % 5) * 0.5;
    const Matrix fast = multiply(left, right), slow = multiplyTextbook(left, right);
    for (std::size_t index = 0; index < fast.data.size(); ++index)
        assert(std::abs(fast.data[index] - slow.data[index]) < 1e-12); // Same arithmetic, different order.
    Matrix rotation(2, 2), point(2, 1);
    const double angle = 0.6;
    rotation.at(0, 0) = std::cos(angle); rotation.at(0, 1) = -std::sin(angle);
    rotation.at(1, 0) = std::sin(angle); rotation.at(1, 1) = std::cos(angle);
    point.at(0, 0) = 3; point.at(1, 0) = -4;
    assert(std::abs(norm(multiply(rotation, point)) - 5.0) < 1e-12); // Orthogonal maps preserve length.
    Matrix columnU(3, 1), rowV(1, 4);
    columnU.data = {1, 2, -1}; rowV.data = {2, 0, 1, 3};
    const Matrix outer = multiply(columnU, rowV); // Rank one: every row is a multiple of rowV.
    for (std::size_t row = 0; row < 3; ++row)
        for (std::size_t column = 0; column < 4; ++column)
            assert(outer.at(row, column) == columnU.at(row, 0) * rowV.at(0, column));
}

What's intentionally missing

Tiling for cache and registers, SIMD inner products, parallelism across rows, a fast path for transposed inputs, views that slice a matrix without copying it, and lower-precision storage. A production GEMM runs to thousands of lines for these reasons, but the arithmetic it performs is the one above.
