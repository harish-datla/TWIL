# The Mask Toolkit


An entity's flags word says it's visible, selectable and on fire. Gameplay code needs to read one of those bits, set another and clear a third without touching the rest. Shifts put a 1 in the right position; masks then act on that position.

A mask is just an integer whose set bits pick the positions you care about. The whole toolkit is four bitwise operators (AND, OR, XOR, NOT) and a couple of idioms, and almost every later trick on this page is built from them[1]. AND keeps only the bits set in both inputs, OR sets any bit set in either, XOR flips the bits where the mask has a 1, and NOT (~) flips every bit. Knuth catalogs the full collection under "Bitwise Tricks and Techniques"[4].
Operation	Expression	What it does
Test bit i	(x >> i) & 1	Isolate one bit as 0 or 1.
Set bit i	x | (1u << i)	Force the bit to 1, leave the rest.
Clear bit i	x & ~(1u << i)	Force the bit to 0.
Toggle bit i	x ^ (1u << i)	Flip the bit.
Low n bits set	(1u << n) - 1	A mask of n ones. Undefined at n = 32: the shift count equals the width.
Extract a field	(x >> shift) & mask	Pull a packed sub-value out.
Isolate lowest set bit	x & -x	Keep only the rightmost 1. The basis of bitset iteration.
Clear lowest set bit	x & (x - 1)	Drop the rightmost 1. Loop on this to visit set bits.

The last two deserve a second look because they fall straight out of two's complement. -x is ~x + 1, and the +1 carries back through the flipped trailing zeros. So -x matches x at the lowest set bit and below, and is inverted everywhere above it. x & -x therefore keeps exactly that bit. Subtracting 1 borrows through the trailing zeros up to the lowest set bit, so x & (x - 1) erases it.

Worked example

Take the byte x = 0b01101000 (104), with bits 6, 5 and 3 set.

-x = ~x + 1 = 0b10010111 + 1 = 0b10011000. AND it with x: only bit 3 survives, so x & -x = 0b00001000 = 8.

x - 1 = 0b01100111. AND it with x: 0b01100000 = 96, which is x with bit 3 cleared.

Keep clearing: 104 → 96 → 64 → 0. Three steps for three set bits.

Together these let you visit the set bits of a word in as many steps as there are set bits, instead of testing all 32 or 64. That's how a bitset is iterated, which is the next section.

[Back to Bit Shifting from Scratch](index.md)
