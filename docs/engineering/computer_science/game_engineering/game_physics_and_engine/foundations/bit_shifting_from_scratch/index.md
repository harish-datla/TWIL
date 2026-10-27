# Bit Shifting from Scratch

An engine packs a color into one integer, checks whether two objects' collision layers match with one AND, and turns a 2D coordinate into a cache-friendly key, all with shifts and masks. The same short lines of bit code also cause bugs that get past senior review: a camera that drifts one unit, but only west of the origin, or a shift that gives 1 on one machine and 0 on another. This page teaches you to read and write shift-and-mask code and to spot those bugs before QA does. Every idea has a live widget, and the listings are in C and Rust side by side.
Time
~70 min
Level
Junior on-ramp; senior review for the traps
Prereqs
You can read C or pseudocode. No prior binary needed; §2 builds it. The later engine sections assume you have seen a game loop.
Hardware
None. A feel for how an integer sits in a register helps in §5 and §16.
◂ Build a Game Engine
Phase 0 · Foundations
Next · C++ Containers ▸
What you'll be able to do by the end

    Read a value's bits in binary and hex, and predict what a left or right shift does to it, for signed and unsigned types.
    Spot the three shift bugs that survive review: a signed divide written as a shift, a shift count that reaches the type width, and the rotate idiom at a count of zero.
    Store flags, collision layers and component sets as bitmasks, and test or combine them without a branch.
    Pack and unpack colors and bitfields, interleave coordinates into Morton keys, and do fractional math in fixed point.
    Know which bit operations the CPU does in one instruction (popcount, count leading or trailing zeros, the BMI sets) and which C++20 functions reach them.


## Sections

- [Why Bits](01-why-bits.md)
- [Binary and Two's Complement](02-binary-and-twos-complement.md)
- [Left and Right Shifts](03-left-and-right-shifts.md)
- [Signed vs Unsigned: SAR and SHR](04-signed-vs-unsigned-sar-and-shr.md)
- [Multiply, Divide, and the Shift Is Faster Myth](05-multiply-divide-and-the-shift-is-faster-myth.md)
- [The Signed-Division Trap](06-the-signed-division-trap.md)
- [Undefined Behavior and the Count-Masking Footgun](07-undefined-behavior-and-the-count-masking-footgun.md)
- [Rotates, and Why C Has No Rotate Operator](08-rotates-and-why-c-has-no-rotate-operator.md)
- [The Mask Toolkit](09-the-mask-toolkit.md)
- [Flags, Bitsets, and Bitboards](10-flags-bitsets-and-bitboards.md)
- [Packing: Color and Bitfields](11-packing-color-and-bitfields.md)
- [Morton Order and Swizzling](12-morton-order-and-swizzling.md)
- [Fixed-Point Arithmetic](13-fixed-point-arithmetic.md)
- [Two Famous Hacks](14-two-famous-hacks.md)
- [Counting Bits: Popcount, CLZ, CTZ](15-counting-bits-popcount-clz-ctz.md)
- [Hardware Bit-Manip: BMI1 and BMI2](16-hardware-bit-manip-bmi1-and-bmi2.md)
- [Pitfalls](17-pitfalls.md)
- [What's Intentionally Missing, and What's Next](18-whats-intentionally-missing-and-whats-next.md)
