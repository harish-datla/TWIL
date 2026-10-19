# Atomics and Fences at the Machine Level


A lock-free job queue runs fine on a PC and fails on an Arm handheld. Reading what each std::atomic operation compiles to on each platform shows why. Here is the short version of the Memory Model tutorial: every lock-free std::atomic operation in C++ lowers to a specific machine instruction whose ordering guarantees match the requested memory_order. The mapping on x86-64[24]:

    Relaxed load / store. Plain mov. x86's TSO memory model already gives every load acquire ordering and every store release ordering, so relaxed and acquire/release emit the same instruction (they still constrain how the compiler may reorder code).
    Acquire load, release store. Also a plain mov, for the TSO reason above.
    Sequentially-consistent load. Still a plain mov; the mapping puts the extra cost on the seq-cst store instead.
    Sequentially-consistent store. Either mov followed by mfence (GCC's historical choice) or xchg [mem], reg (Clang's choice; xchg with a memory operand has an implicit LOCK prefix and is a full barrier). Both keep later loads from completing until the store buffer has drained[25]. The two forms are functionally interchangeable; xchg is usually the cheaper of the two on recent cores.
    Atomic RMW (fetch_add, compare_exchange). A LOCK-prefixed instruction (lock xadd, lock cmpxchg). The LOCK prefix asserts cross-core atomicity on the cache line and acts as a full barrier on x86.

Worked example

std::atomic<int> flag; compiled with GCC 13 and Clang 18 at -O2:

flag.store(1) (seq-cst): mov eax, 1 then xchg [flag], eax from both compilers; GCC dropped its old mov + mfence sequence. flag.store(1, std::memory_order_release): a single mov [flag], 1. flag.load(): a single mov. flag.fetch_add(1): lock xadd.

On Arm (AArch64) the lowering is more visible. An acquire load lowers to LDAR (Load-Acquire Register) and a release store to STLR (Store-Release Register). Seq-cst loads and stores use the same two instructions: ARMv8 defined LDAR/STLR so that, used together, they give sequential consistency without extra barriers[24]. 32-bit ARMv7 had no such instructions and surrounded plain loads and stores with DMB barriers instead; P0668 revised the C++ model partly so that fence-based mappings like those, and the ones used on POWER, are actually correct. The 2013 paper by Lê, Pop, Cohen, and Zappa Nardelli[26] shows what a correct C11-atomics version of a real lock-free structure, the Chase-Lev work-stealing deque, needs on Arm and POWER; see the memory model tutorial for the full picture.

[Back to x86-64 Assembly from Scratch](index.md)
