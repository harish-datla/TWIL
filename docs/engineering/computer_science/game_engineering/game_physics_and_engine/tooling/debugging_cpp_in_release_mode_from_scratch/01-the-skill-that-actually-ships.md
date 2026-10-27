# The skill that actually ships

A crash report arrives from a player's machine. The call stack, the registers and the faulting instruction in it all come from the optimized build, because that's the build players run. The debug build is a development tool, not a shipping target: it's too slow to hit frame rate, too big to fit in console memory, and too forgiving to expose the data races that only show up when the code runs fast. So almost every C++ game that crashes on a player's machine was compiled with optimizations on. Reading that binary directly, instead of reasoning only from the source, is what turns "we couldn't reproduce it" into a fix.

These kinds of bug often show up only under optimization:

    Latent undefined behavior (UB) the optimizer takes seriously. The C and C++ standards give some operations, such as signed integer overflow, no defined meaning at all; -O2 may rewrite or delete a branch that depended on it[1]. -O0 kept the branch and the bug looked benign.
    Data races that the slower debug runtime hid. A data race is two threads touching the same memory, at least one writing, with nothing forcing an order between them. Debug allocators and checked iterators (MSVC's _ITERATOR_DEBUG_LEVEL) add latency at every allocation and container access, which can hide an unsafe interleaving that release timing exposes[2].
    Reads of uninitialized memory. MSVC's debug CRT fills fresh heap allocations with 0xCD, freed heap with 0xDD, and the guard bytes around each allocation ("no man's land") with 0xFD[3]. The 0xCC you see on uninitialized stack comes from the compiler's /RTCs runtime check (on in the default Debug configuration), which fills locals with the int 3 opcode; it isn't the CRT's. Release builds don't fill at all: the memory holds whatever was there before.
    Inlining-dependent bugs. A call the debug build made out of line gets inlined in release, sometimes after devirtualization. The lifetime of a temporary, the stack layout around a local whose address escapes, and the order the compiler picks for unsequenced argument evaluation can all change.
    Stripped or different symbol resolution. A weak symbol resolution that happened to pick the local definition in debug can pick a vendored library's version in release; an inlined-and-elided helper in one translation unit can survive in another.
    Floating-point reassociation. -ffast-math permits the compiler to reorder additions, fuse multiply-adds, treat NaN as impossible, and assume no signed zero[4]. Determinism across -O0 and -O2 goes with it.

Game-specific reasons the release binary is the one that matters: QA and platform certification test the shipping configuration; a memory-budget bug only triggers when the engine pre-allocates the production heap; a 4 ms frame spike from a rare allocator path needs production frame rates to surface. None of those reproduce reliably under -O0.
The shape of a release-only bug

Here's the whole genre in one small function. It's compiled twice by the same compiler, once at -O0 and once at -O2, and called with the same input:
debug ·
-O0 -g

int guard(int n) {
  int doubled = n + n;       // safe at -O0: even on overflow,
  if (doubled < n) return -1; // the compare actually runs
  return doubled;
}
// guard(INT_MAX) returns -1, as the source suggests.

release ·
-O2

int guard(int n) {
  int doubled = n + n;
  if (doubled < n) return -1;
  return doubled;
}
// guard(INT_MAX) hits signed overflow, which is undefined behavior
// per the C standard (C17 §6.5/5). The optimizer assumes n + n
// never overflows, so doubled < n holds exactly when n < 0, and
// GCC and Clang compile the test as n < 0. For INT_MAX that is
// false: the function returns the wrapped sum (-2 on x86-64).

The two binaries disagree on the same source, and neither compiler is wrong. The C and C++ standards leave signed overflow undefined[1], so the optimizer may assume the addition never overflows. Under that assumption n + n < n is equivalent to n < 0, a cheaper test, and the substitution is legal. The overflow check only ever worked through the overflow the standard says can't happen. -O0 ran the comparison literally because the optimizer was off; -O2 ran the equivalent the standard permits. The bug is in the source, and the release build is the one that exposes it.

Worked example

Call guard(INT_MAX), where INT_MAX is 2,147,483,647. Mathematically n + n is 4,294,967,294, which doesn't fit in a 32-bit int, and the x86 add instruction wraps it to 4,294,967,294 − 232 = −2. At -O0 the compare runs as written: −2 < 2,147,483,647 is true, so guard returns −1. At -O2 the compiler tests n < 0 instead: 2,147,483,647 < 0 is false, so guard returns doubled, which is −2. Same input, two answers, and the overflow check never fires in the build you ship.

The same shape recurs throughout this tutorial: a thing the debugger or the source pretends to know turns out to be a fiction the optimizer has already moved past, and the path forward is to read the artifact (the assembly, the shadow-memory map, the unwind table, the minidump) instead.
Check Your Understanding Q1 of 1

A teammate reports a bug that only happens in the shipping build. They have already verified the same input in a debug build does not reproduce. Which is the most useful immediate hypothesis?
There is UB in the source that the optimizer is exercising. Build the same source with the corresponding sanitizer enabled (UBSan or ASan) and the debug runtime disabled, and re-run the failing input.
It is a compiler bug in -O2. File a Clang/MSVC issue.
Disable optimizations for the affected file with #pragma optimize("", off) and ship.
0/1 correct
