# Sanitizers: instrumented runtime checks

A heap corruption crashes one playtest in a few hundred, each time somewhere different, and never under the debugger. A sanitizer is a compiler mode that inserts runtime checks around risky operations (memory accesses, integer arithmetic, thread synchronization) and reports the first violation with a stack trace, at the bad access itself rather than at the crash it causes later. Four are in wide use. Clang ships all four, GCC ships ASan, UBSan and TSan, and MSVC ships only ASan (since Visual Studio 2019 16.9, per Microsoft's docs):

    AddressSanitizer (ASan). Detects out-of-bounds reads and writes on heap, stack, and globals; use-after-free; use-after-return; double-free[6]. The mechanism is a shadow memory: every 8 bytes of program memory map to one byte of shadow that records how many of the 8 are addressable. Every load and store is preceded by a shadow check the compiler inserts inline. The original paper measured an average 1.73× slowdown on SPEC CPU2006, with the worst benchmarks (perlbench, xalancbmk) at 2.60× and 2.67×, and an average 3.37× increase in memory use[6].
    UndefinedBehaviorSanitizer (UBSan). Detects signed integer overflow, shift-by-too-many-bits, null dereference, misaligned pointer use, and a long list of others[5]. Lower overhead than ASan; most checks are a compare and a branch per checked operation.
    ThreadSanitizer (TSan). Detects data races by recording every memory access in shadow memory and checking it against earlier accesses to the same location by other threads, using vector clocks updated at each synchronization operation[27]. The Clang docs give 5× to 15× as the typical slowdown and 5× to 10× as the typical memory overhead. Used on CI machines and in playtests, not in shipping builds; it runs on Linux, macOS, Android and the BSDs, not Windows.
    MemorySanitizer (MSan). Detects reads of uninitialized memory by tracking a bit-precise "is this initialized" shadow, one shadow bit per bit of application memory[28]. Requires every dependency, including the standard library, to be MSan-instrumented; this makes it harder to deploy than ASan, but when you can deploy it, it catches a class of bug nothing else does.

On a game, a common configuration is: ASan + UBSan on every CI build that doesn't need shipping performance; TSan on a separate "race-hunt" build run nightly; MSan only when a specific bug suggests an uninitialized read. A build with -fsanitize=address,undefined turns most of the memory-safety and UB bugs the tests exercise into reports with stack traces, for the cost of a separate build configuration.

Worked example

A 10-minute automated playthrough takes about 17 minutes under ASan at the paper's 1.73× average slowdown, and closer to 27 minutes if your code behaves like its worst benchmarks (2.67×). Memory is often the tighter limit: at the paper's 3.37× average, a game that uses 4 GB needs about 13.5 GB, so plan the test machines for it.

The shadow-memory mechanism is worth understanding because the failure mode of "ASan didn't catch it" is usually traceable to it. Each 8-byte aligned chunk of memory has one shadow byte; the byte's value is 0 (all 8 bytes addressable), 1–7 (only the first k bytes are addressable, the rest belong to a redzone), or a negative value (the entire chunk is poisoned). On every memory access the compiler emits an inline check that loads the shadow byte and traps if the access reads or writes a poisoned chunk.

Worked example

In the widget, malloc(13) returns a pointer to byte 8 of a 32-byte view, which is four 8-byte chunks. Bytes 8 to 15 fill chunk 1, so its shadow byte is 00. Bytes 16 to 20 are the allocation's last 5 bytes, so chunk 2's shadow byte is 05: offsets 0 to 4 are valid. Chunks 0 and 3 are redzones, FA. p[13] is byte 8 + 13 = 21, offset 5 in chunk 2, and 5 isn't below 5, so the check fails. ASan reports it as 0 bytes after the 13-byte region, its way of saying the first byte past the end.

The widget below is a small simulation. Allocate a buffer; the runtime poisons the redzones around it. Read or write past the end of the buffer; ASan traps the access and prints the report. Free the buffer; the entire chunk turns into a use-after-free trap if you touch it again:
Live · AddressSanitizer shadow memory
malloc(13)
write in bounds
write past end
free
use after free
reset
user memory · 32 bytes shown
00
01
02
03
04
05
06
07
08
09
0A
0B
0C
0D
0E
0F
10
11
12
13
14
15
16
17
18
19
1A
1B
1C
1D
1E
1F
shadow memory · one byte per 8 bytes of user memory
FA
FA
FA
FA
addressable redzone poisoned freed
log runtime ready. no allocation yet; heap the allocator hasn't handed out is poisoned (shadow FA).
Click malloc(13), then try the other buttons in order: a write in bounds, a write one past the end, free, and a write after the free. The top grid is 32 bytes of user memory and the row below it is the shadow, one byte per 8-byte chunk. The shadow-byte encoding here mirrors the real AddressSanitizer: a chunk fully addressable is 0, a partially-addressable chunk holds the count of addressable bytes, and a poisoned chunk holds a negative value identifying the kind of poison (FA heap redzone, FD freed, F5 stack after return)[6]. The 13-byte allocation shows the partial encoding: its second chunk reads 05, so p[13], one past the end, is caught even though it shares a chunk with valid bytes. What ASan can't see is an overflow that stays inside one allocation, such as a char name[16] field overrunning into the next field of the same struct: every byte it touches is addressable.
Sanitizers that survive into production

ASan's slowdown and memory cost are fine on a CI machine and out of the question on a player's, yet some memory bugs only show up in the field. Memory tagging moves the check into hardware or near it. HWASan uses AArch64's top-byte-ignore feature to store an 8-bit tag in the high bits of each pointer, with compiler-inserted checks against a matching tag kept in shadow memory for each 16-byte granule. It needs far less memory than ASan (no redzones or quarantine) but is still a testing tool[29]. ARMv8.5's MTE does the same check in hardware with 4-bit tags; Android's documentation recommends its low-overhead asynchronous mode for production use on devices that have it (source.android.com).

GWP-ASan is the one built for shipping builds: it samples a small fraction of allocations at random and routes them through a guard-page allocator, missing the rest but adding negligible overhead and catching overflows and use-after-free on the sampled allocations[30]. It ships by default in Chromium and in Scudo, Android's hardened allocator.

Windows has Page Heap (enabled per-binary with gflags.exe /p /enable game.exe /full), which places each allocation at the end of its own page, followed by an inaccessible guard page[31]. It catches heap overflows and use-after-free with no recompile, at the cost of a much larger memory footprint and a substantial allocator slowdown. Useful when you can reproduce a bug but can't rebuild the binary that reproduces it.
Check Your Understanding Q1 of 1

An ASan build writes 20 bytes into name, the first field of a heap-allocated struct Player { char name[16]; int health; }. ASan reports nothing, yet health is corrupted. Why?
The overflow never leaves the allocation. ASan poisons redzones between objects, not between the fields of one object, so the four extra bytes land in health, which is addressable, and the shadow check passes.
ASan's 8-byte shadow granularity hides overflows smaller than a chunk.
The optimizer removed the check because it could prove the write was in bounds.
0/1 correct
