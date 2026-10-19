# Counting Bits: Popcount, CLZ, CTZ


An allocator keeps one bit per memory slot, set when the slot is free, and needs the first free slot now. A hash table needs to round its size up to the next power of two. A bitboard engine needs to count a side's pieces. All three are one-instruction questions about the bits of a word.

Three queries cover them. The population count (popcount) is how many bits are set. Count leading zeros (clz) is how many zero bits sit above the highest set bit. Count trailing zeros (ctz) is how many sit below the lowest set bit, which is that bit's index. Popcount gives the size of a bitset, and the popcount of a ^ b is the number of positions where two words differ (their Hamming distance). Leading zeros give an integer log base two, which sizes hash tables and buddy allocators. Trailing zeros give the index of the lowest set bit, which is how you iterate a bitset or find the first free slot.

Worked example

x = 40 = 0b101000 in a 32-bit word.

popcount = 2 (bits 5 and 3). ctz = 3: the lowest set bit is bit 3. clz = 26: the highest set bit is bit 5, so bits 31 down to 6 are zero.

Integer log2: 31 − clz = 5, and indeed 25 = 32 ≤ 40 < 64. Rounding 40 up to a power of two gives 64.

Modern hardware does all three in one instruction: AMD shipped POPCNT and LZCNT with Barcelona in 2007[14], Intel picked up POPCNT with SSE4.2 in 2008, and TZCNT came with the BMI extensions[10]. There is a sharp edge: the older bit-scan instructions BSR and BSF leave their result undefined when the input is zero, so clz and ctz of zero need care; LZCNT and TZCNT were defined to return the operand width instead. C++20 wraps all of this in <bit>: std::popcount, std::countl_zero, std::countr_zero, and std::bit_width, each lowering to the hardware instruction when present[12].

When the instruction isn't available, the classic fallback is a SWAR population count ("SIMD within a register"): treat the word as many small lanes and add them in parallel with ordinary integer arithmetic. First add adjacent bits in pairs, giving 16 two-bit counts, then add adjacent pairs of those into nibbles, then bytes, halving the number of lanes each step. Thirty-two bits fall in five such steps, about fifteen instructions, instead of a 32-iteration loop. The technique goes back to HAKMEM, MIT AI Memo 239 from 1972[3], and is tabulated in Hacker's Delight[2]. Trailing-zero counts have a software form too: isolate the lowest set bit, multiply it by a de Bruijn constant (a number built so that each power of two times it leaves a different 5-bit pattern in the top bits), and look those top five bits up in a 32-entry table[7].
Live · Popcount race 0xB7E15163
▶ run SWAR fold
random word
naive loop
17 (32 steps)
SWAR fold
17 (5 steps)
hardware
17 (1 POPCNT)
Each row is the 32-bit word after one SWAR step, from the input at the top to the final count at the bottom, and the three boxes give the count by each method. Press run SWAR fold to watch the partial sums accumulate as the lane width doubles each row, or random word for a new input. The starting word 0xB7E15163 has 17 set bits. The naive loop tests one bit per iteration: 32 steps. The SWAR fold sums in parallel and reaches the same total in 5. The hardware POPCNT does it in a single instruction.

[Back to Bit Shifting from Scratch](index.md)
