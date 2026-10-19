# Hardware Bit-Manip: BMI1 and BMI2


You open a profile of the bitset loop from §10 and the disassembly says blsr where you wrote x & (x - 1). The compiler has replaced your two-step idiom with one instruction from a bit-manipulation extension.

Since 2012–2013, x86 CPUs have shipped two such extension sets, BMI1 and BMI2 (Bit Manipulation Instructions), that turn several of this page's multi-step idioms into single instructions[10]. Compilers emit them automatically when you target a recent enough CPU (for example with -mbmi2 or -march=haswell on GCC and Clang). Recognizing them in a disassembly tells you what the source was doing.
Instruction	Replaces	Meaning
BLSR	x & (x - 1)	Clear the lowest set bit.
BLSI	x & -x	Isolate the lowest set bit.
ANDN	~a & b	AND-NOT in one op, no separate NOT.
BZHI	x & ((1 << n) - 1)	Zero the bits at and above position n.
SHLX / SHRX / SARX	<< / >>	Shift by a variable count without touching flags.
PEXT / PDEP	spread/gather loops	Extract or deposit bits selected by a mask, in parallel.

Two are worth a closer look. The old variable-count shift, shl reg, cl, has to take its count in the CL register. It also leaves the flags register unchanged when the count is 0, so it has to wait for whichever instruction last wrote the flags, a false dependency, even when nothing reads them. SHLX, SHRX and SARX take the count in any register, never touch the flags, and each decodes to a single micro-op, one of the simple internal operations the core actually executes[11]. And PDEP takes the low bits of a value and deposits them, in order, into the positions where a mask has 1s. With the mask 0x55555555 (every even bit) it spreads x into the even positions in one instruction, so the Morton encode from §12 becomes one PDEP per coordinate and an OR.
One caveat before you reach for PEXT

PEXT and PDEP are one micro-op with 3-cycle latency and one-per-cycle throughput on Intel and on AMD from Zen 3 onward, but on AMD's Zen, Zen+, and Zen 2 they are implemented in microcode, with latency and reciprocal throughput around 18 cycles[11]. Code that assumed they were fast (notably some chess engines using PEXT for sliding-piece attack tables) ran dramatically slower on those parts. If you dispatch on PEXT, gate it on the vendor and family, not just the feature bit.

[Back to Bit Shifting from Scratch](index.md)
