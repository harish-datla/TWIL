# Heisenbugs and undefined behavior

You add a printf to see a value, and the crash goes away. Remove it, and the crash comes back. That's a heisenbug: a bug whose presence depends on observation. It can also vanish under the debugger or in the debug build and reappear in release. The category is real, and the usual causes are:

    Optimizer-exposed UB. Signed overflow, strict aliasing, null deref, race on a non-atomic, oversized shift. The optimizer assumes UB doesn't happen and rewrites code accordingly. Adding a printf changes the surrounding code enough that the optimizer's assumption changes too, and the bug moves[1].
    Reads of uninitialized memory. The value read is whatever was at that address, and that depends on every prior allocator call and every prior write. Adding instrumentation changes the allocator state. MSan (or Valgrind's Memcheck) catches these; without one, the value read changes with every unrelated edit.
    Timing-dependent races. The debug runtime adds latency at lock and allocator boundaries, which changes which interleavings happen. A race the debug timing hid can fire in release, and attaching a debugger shifts the timing again.
    Order-of-evaluation differences. C++17 fixed some of the most surprising ones, but function-argument evaluation order is still unspecified: C++17 only stopped arguments from interleaving[34]. f(g(), h()) may call g before or after h; if both have side effects on shared state, the result differs. If g and h each bump a shared counter that starts at 0 and return the new value, f can receive (1, 2) from one compiler and (2, 1) from another.
    Floating-point reassociation. Under -ffast-math, the optimizer fuses, reorders, and treats NaN as impossible. Two builds of the same source can give different bit-exact answers; cumulative drift in physics or AI can become divergent over many frames[4].

The textbook example, walked through in part 2 of Chris Lattner's "What Every C Programmer Should Know About Undefined Behavior" series[35], is a function that dereferences a pointer before checking it for null. The C standard says dereferencing a null pointer is undefined; the compiler is then permitted to assume the pointer is non-null at every later point, which means the subsequent if (p) check is dead code that the optimizer deletes. The same shape shipped as a Linux kernel privilege-escalation bug in 2009 (CVE-2009-1897): a tun-driver patch added sk = tun->sk just above an existing null check on tun, GCC removed the check on the same logic, and the kernel oops became a local-root exploit[35]. The pattern is generic: the optimizer's correctness only requires the source's behavior to match in non-UB executions, and UB in the source is the lever that lets it delete code.

The categories below are worth recognizing on sight because they explain many "it works in debug" reports.
Strict aliasing

You read a float's bits through a uint32_t*, or write through an int* and a float* that point at the same memory. The C and C++ standards say an object may only be read or written through an expression of its own (or a compatible) type, with a few exceptions, the main one being character types like char and unsigned char[13]. The optimizer uses this to assume that int* and float* point to disjoint storage and to reorder loads and stores. The classic type pun, and the classic way it breaks:
strict-aliasing UB · do not write this

float bits_to_float(uint32_t bits) {
  return *(float*)&bits;   // UB: reads a uint32_t object through a float lvalue.
}                              // Often compiles to the expected movd; nothing guarantees it.

int store_then_load(int* count, float* scale) {
  *count = 1;
  *scale = 0.0f;                // may not alias *count, says the rule...
  return *count;               // ...so -O2 can fold this to "return 1" even when
}                              // scale == (float*)count and the store zeroed it.

The compliant equivalents are std::memcpy(&result, &bits, sizeof result) in C++11+ or std::bit_cast<float>(bits) in C++20. Both compile to identical machine code; both are defined; neither relies on the strict-aliasing exception.

Worked example

Say scale points at the same 4 bytes as count. On a little-endian CPU, *count = 1 stores the bytes 01 00 00 00, and *scale = 0.0f stores 00 00 00 00 over them, so memory now holds 0. At -O0 the function reloads *count and returns 0. At -O2 the compiler may assume a float store can't change an int, skip the reload and return 1. The defined way to reinterpret bits is std::bit_cast<float>(0x3F800000u), which gives 1.0f, because 0x3F800000 is the bit pattern of 1.0 in IEEE 754 single precision.
Use-after-move

The C++ standard library's "moved-from" objects are in a "valid but unspecified" state. Many user-defined types follow the same convention. Reading from a moved-from object is not UB by the standard, but it is almost always a bug; the value is whatever the move constructor left behind, which depends on the implementation. clang-tidy's bugprone-use-after-move check and MSVC's code-analysis warning C26800 catch many of these statically.
Iterator invalidation

Calling vector::push_back or vector::insert can reallocate the underlying buffer, invalidating every iterator and pointer into the vector. A range-for loop over a vector that calls a function that pushes back into the same vector is the canonical case. Debug iterators (_GLIBCXX_DEBUG, MSVC's _ITERATOR_DEBUG_LEVEL=2) catch these at runtime; release iterators do not[2].
