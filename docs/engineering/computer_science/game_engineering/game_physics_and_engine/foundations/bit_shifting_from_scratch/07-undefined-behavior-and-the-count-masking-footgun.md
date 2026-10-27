# Undefined Behavior and the Count-Masking Footgun


A bitset helper builds a mask of the low n bits with (1u << n) - 1. It passes every test until someone asks for all 32 bits. On x86 the result is then 0 instead of all ones, and a different compiler, or the same compiler at a different optimization level, can give a different answer from the same line.

That's undefined behavior (UB): code the C and C++ standards place no requirements on. The compiler may assume it never happens and optimize on that assumption, so the program can do anything. Shifts carry a surprising amount of it. In both C and C++ a shift is UB when the count is negative, or when it's greater than or equal to the bit width of the left operand (after small types are promoted to int). C adds a rule for signed left shifts: shifting a negative value, or shifting a positive one so far that the result doesn't fit, is also UB. C++ had a similar rule until C++20, which defines signed left shift as wrapping around modulo 2N[12].

The count rule is the practical landmine, because the hardware disagrees with the language. An x86 shift instruction keeps only the low 5 bits of the count for a 32-bit operand, or the low 6 bits for a 64-bit one, and ignores the rest[10]. 32-bit ARM's register-count shifts use the bottom 8 bits of the count instead. Leaving the case undefined lets a C compiler emit the bare instruction on each machine without checking the count first[13]. The result: one line with a variable shift count can produce three different answers.

Worked example

Shift the 32-bit value 1 left by 32.

x86: 32 in binary is 100000, whose low 5 bits are 0, so the CPU shifts by 0 and the result is 1. A count of 33 masks to 1 and gives 2.

32-bit ARM: 32 fits in 8 bits, so the CPU really shifts by 32 and the result is 0.

A compiler that sees a constant count can fold the expression at compile time, often to 0, or reason that the line can't be reached. In the bitset helper above, x86 turns (1u << 32) - 1 into 1 − 1 = 0.
Live · One shift, three answers 1 & 31 = 1
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
05
04
03
02
11
00
shift count 1
value = 1
value = 0x00FF00FF
value = 0xDEADBEEF
C standard
0x00000002
value << count
x86 hardware
0x00000002
count masked to 5 bits
compile-time fold
0x00000002
constant folded to 0

The bit row is what x86 hardware produces for value << count, and the three columns give the same expression under three rules. Drag the count to 32 or beyond. The C-standard column goes red (undefined), the x86 column wraps the count (the header shows 32 & 31 = 0) and keeps shifting the live value, and a compiler that folds the expression at compile time may simply produce 0. All three are "correct" under their own rules. The takeaway: never let a shift count reach the operand width.
Other languages made different choices

Because the widgets here run in your browser, they are a live example of a third rule. JavaScript coerces operands to 32-bit integers for bitwise ops and masks the shift count to 5 bits, so 1 << 32 is 1, and it has a separate >>> operator for unsigned (logical) right shift. Java does the same masking (& 31 for int, & 63 for long) and also spells logical right shift >>>. None of these are "undefined": they are defined to mask. Of the languages here, only C and C++ leave it undefined. Rust panics on an over-wide count in debug builds and masks it in release builds. Port a shift between these languages without thinking and the count-masking rule will eventually surprise you.
Check Your Understanding Q1 of 1

On a 32-bit unsigned value, what does the expression 1u shifted left by 32 evaluate to in C?
It is UB. The shift count equals the operand width, so the standard places no requirement on the result, and different compilers or optimization levels can yield 1, 0, or something else.
Exactly 0, because all the set bits have been shifted out the top.
Exactly 2 to the 32nd power, promoted to a 64-bit result.
0/1 correct

[Back to Bit Shifting from Scratch](index.md)
