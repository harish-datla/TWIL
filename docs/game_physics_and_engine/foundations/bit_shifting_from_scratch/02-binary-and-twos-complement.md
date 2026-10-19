# Binary and Two's Complement


You stop in the debugger and a health counter that should read −1 shows 4,294,967,295. Nothing is corrupted: the debugger is reading the same 32 bits as an unsigned number. To see why, you need two things: how bits make a number, and how the sign is stored.

Binary is ordinary place value in base two. Counting from the right and starting at 0, bit i is worth 2i: 1, 2, 4, 8, 16 and so on. Long binary strings are hard to read, so you usually write them in hex, where each hex digit stands for exactly four bits (a nibble). Two nibbles make a byte, and four bytes make a 32-bit word.

Worked example

The byte 0b00101101 has bits 5, 3, 2 and 0 set: 32 + 8 + 4 + 1 = 45.

Split it into nibbles: 0010 is 2 and 1101 is 13, written D. So 45 is 0x2D.

The right-most bit, worth 1, is the least significant bit (LSB). The left-most is the most significant bit (MSB). In an unsigned 32-bit integer the MSB is just the 231 place. In a signed one it holds the sign.
Two's complement: how the sign is stored

Nearly every machine you'll ship on stores signed integers in two's complement. The idea is simple: the top bit counts as a negative place value. In 8 bits the top bit is worth −128 instead of +128, and every other bit keeps its usual value. So 0b11111111 is −128 + 127 = −1, not 255. To negate a number you flip every bit and add one: -x == ~x + 1.

Worked example

Negate 45 in 8 bits. Flip every bit of 0b00101101 to get 0b11010010, then add one: 0b11010011.

Read it back: the top bit is worth −128 and the rest add up to 64 + 16 + 2 + 1 = 83, so the byte is −128 + 83 = −45. Read the same byte as unsigned and it's 128 + 83 = 211. That's the debugger's health counter in miniature.

Three consequences fall out of this, and they drive most of the page:

    There is one more negative value than positive: a 32-bit int runs from −231 to 231−1. The most negative value, INT_MIN, has no positive twin, so -INT_MIN overflows.
    The sign lives entirely in the top bit, so a right shift that copies that bit downward keeps negative numbers negative (§4 calls this an arithmetic shift).
    Shifting a signed 32-bit value right by 31 copies the sign bit into every position. The result is all zeros when x isn't negative and all ones (−1) when it is. That all-or-nothing mask is the starting point for many branchless tricks.

C++20 made two's complement the only legal way to store signed integers, so code can rely on it[12]; C23 did the same. Before that, the standards still allowed other encodings in principle, even though no mainstream compiler used them.
Live · Bit inspector
unsigned
signed
031
030
029
028
027
026
025
024
123
122
021
020
019
018
017
016
115
114
113
112
111
110
19
18
17
16
15
04
13
12
11
00
0
all ones
INT_MIN
random

binary
    00000000 11000000 11111111 11101110
hex
    0x00C0FFEE
unsigned
    1,26,48,430
signed
    1,26,48,430
reading
    1,26,48,430 (unsigned)

The row is one 32-bit value, bit 31 on the left (outlined, because it's the sign bit) and bit 0 on the right. Click any bit to toggle it, or press all ones and INT_MIN. The unsigned and signed readings agree until bit 31 is set. Then the unsigned value jumps past two billion and the signed value goes negative: all ones reads as 4,294,967,295 unsigned and −1 signed. Both readings come from the same 32 bits.

[Back to Bit Shifting from Scratch](index.md)
