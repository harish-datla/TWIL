# std::array: Fixed Size, on the Stack


A vertex position is always three floats and a color is always four. Storing them in a std::vector would cost a heap allocation per value for a size that never changes. std::array is the container for that case: a thin wrapper around a C-style array, sized at compile time, with no heap allocation, because all the elements live in the object itself[6]. std::array<float, 3> is exactly three floats in twelve bytes, nothing more. It exists for two reasons: it gives a C array a real STL interface (.begin(), .size(), .data(), etc.), and it makes the size part of the type so the compiler can check it.

Worked example

std::array<float, 3>: 12 bytes, stored wherever the array object is, and no allocation.

std::vector<float> holding the same 3 floats, on 64-bit libstdc++ or libc++: a 24-byte object (three pointers) plus a separate 12-byte heap block, which glibc's malloc serves from a 32-byte chunk. That's 56 bytes and an allocator call to hold 12 bytes of data.
array_basics.cpp · what's actually in memory

// Three floats, allocated wherever the std::array object lives.
// If 'positions' is a local, it's on the stack; if it's a member of a class
// allocated on the heap, it lives in that class's heap block. No
// separate allocation either way.
std::array<float, 3> positions = {0.0f, 1.0f, 2.0f};

// sizeof(positions) is exactly 12. There's no length field; the size is
// encoded in the type so the compiler always knows it.
static_assert(sizeof(positions) == 3 * sizeof(float));

// The interface is std-container-shaped: iterate, index, ask the size.
for (float& element : positions) element *= 2.0f;

// data() returns a pointer to the first element, the same way a C array decays.
// Useful when calling C APIs (OpenGL, Vulkan, sockets) that want a raw pointer.
glBufferData(GL_ARRAY_BUFFER, sizeof(positions), positions.data(), GL_STATIC_DRAW);

Use it whenever the size is a compile-time constant and the container is small enough that copying it by value is cheap. Vertex positions, RGBA colors, 4×4 matrix rows, hash digests, fixed-channel audio mixes, lookup tables of known size. Anywhere a C programmer would have written float v[3];.
When std::array is the wrong answer

The size is part of the type. std::array<int, 3> and std::array<int, 4> are different types and you can't assign one to the other. If the size of the data is "small but not fixed" (usually up to 16 entries, occasionally more), reach for a vector with a small-buffer optimization, such as LLVM's SmallVector[15] or EASTL's fixed_vector, rather than std::array with the maximum size and a hand-maintained length counter. If the cap is hard, C++26's std::inplace_vector is the standard version of that array-plus-count.

[Back to C++ Containers](index.md)
