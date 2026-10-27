# Left and Right Shifts


You have a packed color and want its top byte, or you want the bit that stands for collision layer 5. Both jobs start by moving bits sideways, and that's what a shift does.

A shift slides every bit of a value the same number of places. Left shift (<<) moves bits toward the MSB and feeds zeros in at the bottom. Each place doubles the value, because every bit moves up to the next power of two. Right shift (>>) moves bits toward the LSB and throws away the bits that fall off the bottom. For an unsigned value it feeds zeros in at the top, so each place halves the value, rounding down.

Worked example

Start from 45 (0b101101).

45 << 1 = 0b1011010 = 90, and 45 << 3 = 45 × 8 = 360.

45 >> 1 = 0b10110 = 22: the 1 in bit 0 fell off, so 22.5 rounds down. 45 >> 3 = 0b101 = 5, which is 45 ÷ 8 = 5.625 with the fraction dropped.

So x << 3 is x × 8 and x >> 3 is roughly x ÷ 8. The "roughly" matters for negative values: a right shift has two modes, covered in the next section, and they only differ when the sign bit is set.
Live · Shift playground
logical >>
arithmetic >>
031
030
029
028
027
026
025
024
023
022
021
020
019
018
017
016
015
014
013
012
011
010
09
08
07
06
15
04
13
12
01
10
« x << 1
x >> 1 »
load 45
load −20
load 2³⁰

hex
    0x0000002D
value
    45 / 45 signed

Shift left to multiply by two, right to divide. Load a negative value, then flip the right-shift mode.
The row shows the current value's 32 bits, and the readout gives it as unsigned and as signed. It starts at 45: press x >> 1 and it becomes 22, with the message naming the bit that fell off. Load 2³⁰ and shift left twice: the 1 moves into bit 31 and then off the top, silently, leaving 0. For a signed value that overflow is why C makes pushing a bit into or past the sign bit undefined behavior (§7); C++20 defines it as wrapping. Now load −20 with the right shift set to logical and shift right once: the "divide by two" gives 2,147,483,638 instead of −10, because the sign bit got a zero instead of a copy of itself. Load −20 again, switch to arithmetic, and the same shift gives −10.

[Back to Bit Shifting from Scratch](index.md)
