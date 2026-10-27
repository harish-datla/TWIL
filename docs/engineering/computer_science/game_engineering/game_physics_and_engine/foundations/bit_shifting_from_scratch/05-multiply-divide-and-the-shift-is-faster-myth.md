# Multiply, Divide, and the Shift Is Faster Myth


A reviewer asks you to change index * 8 to index << 3 "for speed". On early microprocessors with slow or missing multipliers that was good advice. On the CPUs games ship on today it changes nothing, because the compiler already emits the shift (or an equivalent lea) for you.

Swapping an expensive operation for a cheaper one that gives the same answer is called strength reduction, and optimizing compilers have done it for decades. Two facts retire the myth. First, a modern CPU shifts with a barrel shifter, a circuit that shifts by any distance in one step, so shifting by 1 and by 31 cost the same. Second, on a modern x86 core an integer multiply takes about 3 cycles before its result is ready, and the core can start a new one every cycle; a shift takes 1 cycle[11]. That gap is real in a long chain of multiplies that each wait for the last, but closing it is the compiler's job, and the disassembly below shows it doing so.
Where "big shifts are slow" came from

Worked example

x * 7 becomes lea eax, [rdi*8] then sub eax, edi: 8x − x. For x = 5 that's 40 − 5 = 35.

Unsigned x / 7 becomes a multiply by a fixed-point reciprocal, 613,566,757 (0x24924925), plus a fix-up. For x = 100: the product is 61,356,675,700; its top 32 bits are t = 14. Then (100 − 14) >> 1 = 43, t + 43 = 57, and 57 >> 2 = 14, which is 100 / 7 rounded down. No divide instruction runs.
Live · What the compiler actually emits
x * 8
x << 3
unsigned x / 8
signed x / 8
x * 7
unsigned x / 7

lea    eax, [0+rdi*8]   ; one address-gen, scaled by 8
ret

The multiply never appears. The compiler folds ×8 into a scaled lea (or a shl eax, 3). Identical machine code to writing x << 3.
Each button shows the x86-64 code an optimizing compiler emits for that expression, with a note underneath. Click x * 8 and then x << 3: the instructions are identical, so choosing one over the other says something about intent and nothing about speed. Then compare unsigned x / 8 with signed x / 8: division is where shifts really save cycles, and the signed case hides a trap. The listings are GCC 14 and Clang 20 at -O2 (the two emit the same instructions for these cases, give or take instruction and register order). Other compilers make different choices for the same job: MSVC keeps an imul for x * 7 and builds the signed-division bias with cdq and and.

So when should you write a shift? When the value is a bit pattern, not a number. color >> 24 says "pull out the alpha byte." 1u << layer says "the bit for this layer." Writing those as color / 16777216 or pow(2, layer) would be obfuscation. Use the shift to signal bit work; use * and / for arithmetic and trust the compiler to reduce it.

[Back to Bit Shifting from Scratch](index.md)
