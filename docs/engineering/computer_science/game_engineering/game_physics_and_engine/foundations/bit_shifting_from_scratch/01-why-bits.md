# Why Bits


Your input code needs to know which of a dozen buttons are held this frame. You could keep twelve bools. Or you could keep one integer and give each button one bit of it: say bit 3 for jump, bit 2 for crouch, and so on. Checking a button is then a single AND instruction, and copying, comparing or clearing the whole set is a single integer operation.

That works because an integer in a register is just 32 or 64 bits, each a 0 or a 1, and the CPU has single-cycle instructions that act on those bits directly. It can shift them sideways, and it can combine two integers bit by bit with AND, OR and XOR. Reading the bits as a number is a convention on top. Bit manipulation works underneath it.

Worked example

Say jump is bit 3 and crouch is bit 2. This frame the input word is 0b1010 (10): bits 1 and 3 are set.

Is jump held? buttons & (1 << 3) is 0b1010 & 0b1000 = 0b1000 (8). Not zero, so yes.

Is crouch held? 0b1010 & 0b0100 = 0, so no. Each test is one AND and one compare, with no branch on the button's index.

In a game engine this kind of code runs constantly, usually in the hottest loops:

    Flags and layers. Which input buttons are down, which collision layers an object belongs to, which components an entity has. One integer, one bit per option, tested with a single AND.
    Packing. An RGBA color is four bytes shifted into one 32-bit word. A draw-call sort key packs material, depth, and pass into one integer so a single integer sort orders the whole frame[16].
    Spatial keys. Interleaving the bits of an (x, y, z) coordinate produces a Morton code, the address scheme behind texture swizzling, linear octrees, and GPU bounding-volume builders.
    Cheap math. Multiply and divide by powers of two, fixed-point fractions, fast reciprocal-square-root seeds, and shift-based random number generators.
    Counting. How many bits are set (the population count), and where the lowest set bit is (the count of trailing zeros). These two drive bitset iteration and free-slot search in allocators.

The first half of the page builds the tools: binary, the two kinds of right shift, and the bugs each one invites. The second half spends them on engine work.

[Back to Bit Shifting from Scratch](index.md)
