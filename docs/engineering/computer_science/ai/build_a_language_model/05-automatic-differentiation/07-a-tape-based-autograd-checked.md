# A tape-based autograd, checked

These complete programs implement reverse-mode autodiff on a tape. Each node is appended in creation order with its value, its inputs and its local derivative toward each input, and the backward pass walks the tape in reverse, accumulating adjoints. The checks compare the gradients of the first section's expression and of the reused-value expression from the third section with centered finite differences. Then they run a small gradient-descent loop on the tape and assert that it converges. Both listings use the same names so you can read them side by side.
Scalar reverse-mode tape, complete programs
C++20
Rust 2021

#include <cassert>
#include <cmath>
#include <vector>
struct Node {
    double value;
    double adjoint;        // ∂output/∂this, filled in by backward().
    int parents[2];        // Tape indices of the inputs; always smaller than this node's index.
    double locals[2];      // ∂this/∂parent for each parent, computed at forward time.
    int parentCount;
};
struct Tape {
    std::vector<Node> nodes;
    int record(double value, int parentA, int parentB, double localA, double localB, int parentCount) {
        nodes.push_back({value, 0.0, {parentA, parentB}, {localA, localB}, parentCount});
        return static_cast<int>(nodes.size()) - 1;
    }
    int leaf(double value)      { return record(value, -1, -1, 0, 0, 0); }
    int add(int a, int b)        { return record(nodes[a].value + nodes[b].value, a, b, 1, 1, 2); }
    int mul(int a, int b)        { return record(nodes[a].value * nodes[b].value, a, b, nodes[b].value, nodes[a].value, 2); }
    int sine(int a)              { return record(std::sin(nodes[a].value), a, -1, std::cos(nodes[a].value), 0, 1); }
    int relu(int a)              { const bool active = nodes[a].value > 0; return record(active ? nodes[a].value : 0, a, -1, active ? 1 : 0, 0, 1); }
    int square(int a)            { return record(nodes[a].value * nodes[a].value, a, -1, 2 * nodes[a].value, 0, 1); }
    // Reverse pass: seed the output with 1, then visit nodes in reverse creation order, which is a
    // topological order, and accumulate (never assign) each node's contribution into its parents.
    void backward(int output) {
        for (Node& node : nodes) node.adjoint = 0;
        nodes[output].adjoint = 1;
        for (int index = output; index >= 0; --index)
            for (int slot = 0; slot < nodes[index].parentCount; ++slot)
                nodes[nodes[index].parents[slot]].adjoint += nodes[index].locals[slot] * nodes[index].adjoint;
    }
};
// f(a, b) = (a·b + sin a)·b, built fresh on a tape each call; returns the output index.
int buildExpression(Tape& tape, int& a, int& b, double valueA, double valueB) {
    a = tape.leaf(valueA); b = tape.leaf(valueB);
    return tape.mul(tape.add(tape.mul(a, b), tape.sine(a)), b);
}
double expression(double a, double b) { return (a * b + std::sin(a)) * b; }
double fanOut(double x, double w) { return std::fmax(0.0, x * w) * w + x * w; }
int main() {
    const double step = 1e-6;
    Tape tape; int a, b;
    const int output = buildExpression(tape, a, b, 1.5, -2.0);
    tape.backward(output);
    const double numericalA = (expression(1.5 + step, -2.0) - expression(1.5 - step, -2.0)) / (2 * step);
    const double numericalB = (expression(1.5, -2.0 + step) - expression(1.5, -2.0 - step)) / (2 * step);
    assert(std::abs(tape.nodes[a].adjoint - numericalA) < 1e-6 && std::abs(tape.nodes[b].adjoint - numericalB) < 1e-6);
    Tape reuse;
    const int x = reuse.leaf(1.2), w = reuse.leaf(0.8);
    const int y = reuse.add(reuse.mul(reuse.relu(reuse.mul(x, w)), w), reuse.mul(x, w)); // w is used three times.
    reuse.backward(y);
    const double numericalW = (fanOut(1.2, 0.8 + step) - fanOut(1.2, 0.8 - step)) / (2 * step);
    assert(std::abs(reuse.nodes[w].adjoint - numericalW) < 1e-6); // Accumulation handles the fan-out.
    double weight = 0.0; // Minimize (3·weight − 6)² by gradient descent on the tape; the optimum is weight = 2.
    for (int iteration = 0; iteration < 200; ++iteration) {
        Tape loop;
        const int parameter = loop.leaf(weight);
        const int loss = loop.square(loop.add(loop.mul(loop.leaf(3.0), parameter), loop.leaf(-6.0)));
        loop.backward(loss);
        weight -= 0.05 * loop.nodes[parameter].adjoint; // A fresh tape per iteration: the graph is rebuilt as the program runs.
    }
    assert(std::abs(weight - 2.0) < 1e-9);
}

What's intentionally missing

Tensor-valued nodes with matrix derivative rules, broadcasting, gradient-of-gradient (the tape would need to record the backward pass itself), saved-tensor release after use, checkpointing, in-place operation tracking, and any parallelism. A production engine spends most of its code on those; the reverse walk at its center is the loop above.
