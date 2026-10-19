# How the Same Atomic Lowers on x86 and ARM


You loosened an ordering in a hot loop, or a reviewer asked you to tighten one, and you want to know what it changes in the machine code. The quickest way to see what each ordering costs is to read what the compiler emits. The table shows GCC 14.2 at -O2 on x86-64 and AArch64; the ARMv8.0 column is built with -mno-outline-atomics and the LSE column with -march=armv8.1-a. Clang 19 emits the same instruction sequences. You can reproduce all of it on godbolt.org:
Operation	x86-64	AArch64 (ARMv8.0)	AArch64 (ARMv8.1+ with LSE)
load(relaxed) 	MOV eax, [counter] 	LDR w0, [counter] 	LDR w0, [counter]
load(acquire) 	MOV eax, [counter] 	LDAR w0, [counter] 	LDAR w0, [counter] (LDAPR on ARMv8.3+ targets)
load(seq_cst) 	MOV eax, [counter] 	LDAR w0, [counter] 	LDAR w0, [counter]
store(relaxed) 	MOV [counter], edi 	STR w0, [counter] 	STR w0, [counter]
store(release) 	MOV [counter], edi 	STLR w0, [counter] 	STLR w0, [counter]
store(seq_cst) 	XCHG edi, [counter] (older GCC: MOV + MFENCE) 	STLR w0, [counter] 	STLR w0, [counter] (LSE changes only RMWs)
fetch_add(relaxed) 	LOCK XADD [counter], eax 	LDXR; ADD; STXR loop 	LDADD
fetch_add(seq_cst) 	LOCK XADD [counter], eax 	LDAXR; ADD; STLXR loop 	LDADDAL
compare_exchange_strong(seq_cst) 	LOCK CMPXCHG [counter], edx 	LDAXR; CMP; B.NE done; STLXR; CBNZ retry 	CASAL
atomic_thread_fence(seq_cst) 	MFENCE (Clang) or LOCK OR [rsp], 0 (GCC) 	DMB ISH 	DMB ISH

LSE arrived in ARMv8.1-A and adds CAS, SWP, and the LDADD family as single instructions[32]. Apple's M1 and later, Cortex-A55/A75 and later, and Neoverse N1 and later have it; older cores such as the Cortex-A57 in the original Nintendo Switch and the Cortex-A53 don't, and use the LL/SC loop. Building with -march=armv8.1-a (or a -mcpu that implies it) emits LSE directly. For older baseline targets, GCC 10+ and Clang default to -moutline-atomics on AArch64 Linux, which turns each RMW into a call to a small helper that uses LSE when the CPU has it and the LL/SC loop otherwise.

Worked example

§11's push makes three atomic accesses: a relaxed load of writeIndex, an acquire load of readIndex, and a release store of writeIndex.

On x86-64 that's MOV, MOV, MOV: nothing beyond plain loads and stores. On AArch64 it's LDR, LDAR, STLR.

Leave every ordering at the seq_cst default instead and x86-64 emits MOV, MOV, XCHG, a store-buffer drain on every push, while AArch64 emits LDAR, LDAR, STLR, where the next push's LDAR has to wait for this push's STLR.

What the table shows:

    On x86, ordering mostly costs nothing. Loads and stores are plain MOVs at every ordering except a seq_cst store, which becomes an XCHG that drains the store buffer. RMWs are LOCK-prefixed at every ordering. So relaxed and acquire/release differ by zero instructions; what they still change is how far the compiler may move surrounding code.
    On ARM, ordering picks the instruction. Relaxed is plain LDR/STR; acquire and release switch to LDAR/STLR; seq_cst loads and stores use those same two instructions and pay for the rule that an LDAR can't pass an earlier STLR. Choosing the weakest correct ordering shows up in the instruction stream on ARM in a way it doesn't on x86.
    On pre-LSE ARM, every RMW is an LL/SC loop. fetch_add becomes a load-exclusive/store-exclusive loop that retries whenever another core touches the line in between, so it degrades under contention. ARMv8.1's single-instruction RMWs complete without retries, which Arm cites as the reason for LSE on many-core systems[32].


[Back to The C++ Memory Model from Scratch](index.md)
