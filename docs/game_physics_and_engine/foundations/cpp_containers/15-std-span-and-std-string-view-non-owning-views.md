# std::span and std::string_view: Non-Owning Views


A draw function takes the vertices to draw. If it takes a std::vector<Vertex> by value, every call copies the mesh; if it takes a const std::vector<Vertex>&, a caller holding a std::array or a raw buffer has to copy into a vector first. Two additions, std::string_view (C++17) and std::span (C++20), fix that. They aren't really containers, but they're how you pass containers around without copying. They are both views: a pointer plus a size, owning nothing, valid only as long as the underlying storage outlives them[28].

    std::span<T> (C++20). A view of a contiguous range of T. Construct it from a std::vector<T>, a std::array<T, N>, a C array, anything contiguous. The function on the other side gets a uniform interface to all of them.
    std::string_view (C++17). The same idea, specialized for character ranges. Has the methods that make string handling pleasant (find, substr, starts_with, ends_with) without owning the bytes.

Worked example

A mesh of 10,000 vertices at 32 bytes each is 320 KB.

Pass it as a std::vector<Vertex> by value: a 320 KB allocation and copy on every call.

Pass it as a std::span<const Vertex>: a pointer and a count, typically 16 bytes on a 64-bit platform, small enough to travel in two registers.

Use them as parameter types whenever a function takes a contiguous range and doesn't need to extend its lifetime. The function gets to work with any contiguous source without overloading on the container type, and the caller doesn't pay for a copy:
span_param.cpp · the right shape for read-only sequence parameters

// Old way: one overload per container type, each with a copy if you
// took the parameter by value, or coupling to a specific container if
// you took it by reference. Both unsatisfying.
void drawTriangles(const std::vector<Vertex>& vertices);
void drawTriangles(const std::array<Vertex, 64>& vertices);
void drawTriangles(const Vertex* vertices, size_t count);

// Modern way: one function, takes a view, the caller passes whatever
// they have. No copies, no overload set, no error-prone count parameter.
void drawTriangles(std::span<const Vertex> vertices);

// Call sites work uniformly:
drawTriangles(meshVertexVector);             // vector → span
drawTriangles(framePushArray);               // std::array → span
drawTriangles({rawPtr, count});              // raw pointer + count → span

Views don't own anything

A view is a pointer plus a length. The underlying storage has to outlive the view, or the pointer dangles. A classic std::string_view bug is constructing it from a temporary std::string and keeping the view past the end of the statement that created the temporary. Clang's -Wdangling warnings (on by default, driven by [[clang::lifetimebound]] annotations and built-in knowledge of view types) and MSVC's code-analysis warnings C26815 and C26816 catch many of these at compile time; turn them on.

[Back to C++ Containers](index.md)
