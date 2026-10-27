# x86-TSO vs ARM


A job system passes every stress test on your x86 PC and the PS5 and Xbox Series builds (both x86), then corrupts state on the Switch build, which is ARM, from the same source. The difference is the hardware memory model: two families cover most of the CPUs games and servers run on, both relax sequential consistency, and ARM relaxes it much further. Each relaxation takes a specific fence to recover:
x86-TSO (Total Store Order)

Sewell et al. 2010[2] give a model, consistent with the vendor manuals and tested against Intel and AMD hardware, in which each core behaves as if it had a FIFO store buffer between it and a single shared memory. The model permits one relaxation from SC:

    Loads can move ahead of earlier stores to a different address (the store-buffer effect). This is the Dekker case from §1.

Everything else holds. Loads aren't reordered with earlier loads, and stores aren't reordered with earlier stores or earlier loads. All cores other than the writer see a store at the same moment, when it drains from the buffer to shared memory, so they all agree on the order of stores (multicopy atomicity). That's why atomic loads and stores are cheap on x86: a plain load already has acquire semantics and a plain store already has release semantics, so only a seq_cst store needs extra work (an XCHG, or a MOV plus MFENCE, x86's full fence) to drain the store buffer. Read-modify-writes are LOCK-prefixed at every ordering and cost the same whichever you pick.
ARMv8-A and POWER (weak memory models)

Maranget, Sarkar, and Sewell[9] document ARMv7 and POWER as non-multicopy-atomic: a store can become visible to some cores before others. The relaxations from SC that both architectures permitted:

    Loads can move ahead of earlier loads (no implicit load-load ordering).
    Stores can move ahead of earlier stores (no implicit store-store ordering).
    Stores can move ahead of earlier loads. The litmus test for this is load buffering (LB): each thread loads one variable and then stores to the other, and both loads can see the other thread's later store.
    Loads can move ahead of earlier stores (as on x86).
    Two readers can see two writers' stores in different orders (IRIW). POWER still permits this; ARMv8 was revised to be multicopy atomic[35], which closes IRIW when the reads are ordered but leaves the four reorderings above in place.

ARMv8-A added acquire and release as instructions: LDAR (load-acquire) and STLR (store-release)[10]. Each orders one direction: no later load or store can move ahead of an LDAR, and no earlier load or store can move past an STLR. An LDAR also can't move ahead of an earlier STLR, which is what lets the same two instructions implement seq_cst. They're cheaper than the DMB ISH full barrier ARMv7 code needed, which orders every earlier load and store before every later one, and C++ memory_order_acquire and memory_order_release lower to them directly.

The widget below shows which litmus-test outcomes SC, x86-TSO, and ARMv8 allow when every access is a plain load or store with no barriers. Comparing them side by side shows exactly which x86 habits break on ARM. The verdicts follow the formal models in the papers above[2][9][35]; they say what's allowed, not how often a given chip produces it:
Live · Litmus tests under SC, x86, ARM
SC
x86-TSO
ARMv8
Each box is one litmus test with its possible outcomes, marked allowed or forbidden. Switch between SC, x86-TSO and ARMv8 and watch which red rows turn green. SC forbids all four surprising outcomes. x86-TSO allows only SB's (0, 0). ARMv8 allows all four with plain loads and stores; its IRIW disagreement comes from reordering each reader's two loads, and goes away once the reads are acquire loads. The playground in §17 lets you change the orderings and see which outcomes each change removes.
Check Your Understanding Q1 of 2

A multithreaded program runs correctly on your x86 dev box but crashes intermittently when you ship to a PS5 (AMD Zen 2, x86) or Xbox Series X (AMD Zen 2, x86) build. Same code, same compiler. What is the most likely cause?
PS5 has a weaker memory model than x86.
A different scheduler caused the threads to interleave in a way the dev box hadn’t exercised. The race exists on x86 too; you just didn’t hit it.
The compiler optimized differently for AMD vs Intel.
0/2 correct

[Back to The C++ Memory Model from Scratch](index.md)
