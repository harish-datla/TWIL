# Shuffles, Blends, Broadcasts: Rearranging the Lanes


Your vertex data arrives as x, y, z, w per vertex but the kernel wants all the x values in one register. Or a debug overlay needs 16 bytes turned into 32 hex characters. Both are lane rearrangement. Shuffles rearrange lanes within a register or across two registers. They get data into the lane order a kernel needs, and when a kernel runs slower than its arithmetic suggests, a shuffle bottleneck is a common cause. The main members of the family:
Mnemonic	What it does	Use it for
vbroadcastss	Replicate one float across every lane.	Splat a scalar into a vector for an FMA: "every lane gets the same deltaTime."
vshufps	Within each 128-bit half, pick the low two output lanes from the first source and the high two from the second, controlled by an 8-bit immediate (a constant inside the instruction).	Transpose helpers; broadcast one component of a vec4.
vpermps	Eight output lanes, each picked from any of the eight input lanes via an index vector.	Arbitrary lane permutation across a whole 256-bit register (AVX2 and later).
vpshufb	Sixteen output bytes, each picked from the 16 table bytes by an index byte; the AVX2 form does this separately in each 128-bit half.	16-entry parallel lookup table per lane; UTF-8 validation; hex encoding; small case-table dispatch[4].
vblendps / vblendvps	Pick lanes from two sources based on an immediate or a per-lane mask vector.	Branchless conditional select (§8); merging two halves of a reduction.
vmovmskps / vpmovmskb	Extract the sign bits of each lane into a scalar integer.	"Branch on whether any lane satisfied the predicate"; convert a SIMD comparison into a scalar test.
vperm2f128	Swap or duplicate the two 128-bit halves of a 256-bit register.	Crossing the 128-bit lane barrier; merging two halves of a reduction.

For non-arithmetic work, the most useful of these is pshufb[4]. It performs a parallel byte-granularity lookup: for each output byte position i, the output is table[indices[i] & 0x0F], or zero if the index byte's top bit is set, where table is the first operand (a 16-byte table) and indices is the second. One instruction, sixteen 16-entry lookups in parallel. Shipping code uses it for UTF-8 validation (Keiser & Lemire 2021[16]), base64 encode/decode (Muła & Lemire 2018[17]), hex conversion, and per-nibble population counts, among many other workloads.

Worked example

Count the set bits in the byte 0xB7, binary 1011 0111. Its low nibble is 7, and the popcount table holds 3 at index 7. Its high nibble is 11 (0xB), and the table holds 3 there too. 3 + 3 = 6, the right count.

After a shift and an AND split every byte into its two nibbles, one pshufb looks up the low nibbles of 16 bytes at once, a second looks up the high nibbles, and one add finishes all 16 bytes.

The widget shows one pshufb. Pick a table; the arrows trace each output byte back to the table entry its index selects:
Live · pshufb parallel lookup table
Nibble → hex digit
Nibble popcount
Identity table
The top row is the 16-entry table, the middle row the index each output byte uses, and the bottom row the results; curves link each output to the entry it picked. Switch tables: the indices stay put and only the outputs change. With the hex table, index 12 picks C and index 10 picks A. A single vpshufb xmm0, xmm1, xmm2 instruction: sixteen lookups in one μop, with 1-cycle latency on Skylake and on Zen 2, 3 and 5 (2 cycles on Zen 4)[1]. The popcount table is the core of the SSSE3 bit-count trick: look up the low and high nibble of every byte and add[4]. The 256-bit AVX2 form (vpshufb ymm) operates on the two 128-bit halves independently; indices in the upper half can only address the upper half's table. The 512-bit AVX-512 form is the same, four-way independent. vpermb, from the AVX-512 VBMI (Vector Byte Manipulation Instructions) extension, is the cross-lane version: each output byte can come from any of the 64 input bytes. VBMI arrived on Cannon Lake (2018) and shipped broadly from Ice Lake (2019) and on AMD from Zen 4.

Two reliable rules about shuffles:

    In-lane shuffles are cheap; cross-lane shuffles cost more latency. AVX's 256-bit register is two 128-bit "lanes" stitched together. vshufps, vpshufb, vpshufd all operate on each 128-bit lane independently. vpermps and vperm2f128 cross the lane boundary; on Skylake they have 3-cycle latency against 1 cycle for the in-lane shuffles[1]. On Skylake every shuffle of either kind issues on port 5 alone, one per cycle, so a shuffle-heavy loop is port-5-bound regardless. The fastest 8-element transpose uses as few cross-lane shuffles as it can.
    Movmask is the bridge from vector to scalar. When you need to branch on whether any (or every) lane satisfied a predicate, vmovmskps extracts the sign bits into a 32-bit integer; tzcnt or popcnt on that integer answers "which lane was the first match" or "how many matched." This is the pattern for early-exit SIMD search loops: SIMD-compare a 256-bit chunk, extract the mask, branch on it[4].


[Back to SIMD from Scratch](index.md)
