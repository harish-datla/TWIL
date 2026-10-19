# Rotates, and Why C Has No Rotate Operator


A hash table with 1,024 buckets picks a bucket from the low 10 bits of a key's hash. If the hash is weak and the keys differ only in their high bits (IDs that count up in steps of 65,536, say), every key lands in bucket 0. A good hash mixes high bits down into the low ones without losing any, and a rotate is one of its standard tools for that.

A rotate is a shift that wraps around: bits that fall off one end come back in at the other. Nothing is lost, so the number of set bits never changes. CPUs have had rotate instructions for decades (ROL and ROR on x86). Hash functions, cryptographic code and pseudo-random number generators (PRNGs) all lean on them.

Worked example

The widget below starts with the 16-bit value 1011 0000 0000 0110, which has 5 set bits.

Rotate left by 1: every bit moves up one place and the top 1 wraps around to bit 0, giving 0110 0000 0000 1101. Still 5 set bits.

Shift left by 1 instead: the top 1 falls off and a 0 comes in at the bottom, giving 0110 0000 0000 1100. Now 4.

C and C++ have no rotate operator, so for years people wrote the idiom (x << n) | (x >> (32 - n)): shift left, shift the bits that fell off back in from the right, and OR the two halves. That idiom has a bug: when n is 0, the second shift becomes x >> 32, which is the UB from §7. The safe version masks the complement: (x << n) | (x >> ((-n) & 31))[22]. GCC 14 and Clang 20 compile both forms to a single ROL at -O2; MSVC 19.4x recognizes the naive form but emits two shifts and an OR for the masked one. Since C++20, std::rotl and std::rotr from <bit> end the guesswork: they are defined for every count, including 0 and negative counts[12].
Rotate, three ways
C / C++
Rust

#include <bit>     // C++20
#include <cstdint>

// The classic idiom. Correct ONLY for 1 <= n <= 31.
uint32_t rotl_naive(uint32_t value, int n) {
    return (value << n) | (value >> (32 - n));   // UB when n == 0
}

// The safe hand-written form: mask the complement so n == 0 works.
uint32_t rotl_safe(uint32_t value, int n) {
    return (value << (n & 31)) | (value >> ((-n) & 31));
}

// The right answer in C++20: defined for every n, compiles to ROL.
uint32_t rotl_std(uint32_t value, int n) {
    return std::rotl(value, n);
}

Live · Rotate ring (16-bit)
rotate
shift
↺ rotate left
rotate right ↻
The ring is a 16-bit word with bit 0 on the right and bit numbers increasing counter-clockwise; the center shows the value in binary and its set-bit count. In rotate mode, press either button as often as you like: the count stays at 5, because bits leaving the top reappear at the bottom. Flip to shift mode and shift repeatedly: the word bleeds to zero as bits fall off the end. A rotate only rearranges the bits, so it loses no information, which is why hashes and PRNGs use it to carry high bits down and low bits up.

[Back to Bit Shifting from Scratch](index.md)
