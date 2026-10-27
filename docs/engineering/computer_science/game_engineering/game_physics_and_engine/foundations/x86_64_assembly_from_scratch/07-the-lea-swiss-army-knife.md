# The lea Swiss Army Knife


You'll see lea all over code that has nothing to do with addresses, doing multiplies by 3, 5 and 9. lea (Load Effective Address) computes a memory address and writes it to a register without dereferencing. Because the addressing mode is the same as the one mov uses, lea can be used as a fast three-operand arithmetic instruction: any value that fits the base + index*scale + displacement form can be computed in one instruction without clobbering the source registers and without touching the flags. The compiler uses this constantly:
lea_tricks.s

; a + b without an add, and without trashing flags or either input
lea  rax, [rdi + rsi]

; 5 * x in one instruction (4*x + x)
lea  rax, [rdi + rdi*4]

; 9 * x  →  (8*x + x), via lea with scale=8
lea  rax, [rdi + rdi*8]

; Address arithmetic: &array[index] for an array of 4-byte ints
lea  rax, [rdi + rsi*4]

; Combined: 3*x + 7 in one instruction
lea  rax, [rdi + rdi*2 + 7]

Worked example

lea rax, [rdi + rdi*2 + 7] with rdi = 5: 5 + 5 × 2 + 7 = 22, which is 3 × 5 + 7. One instruction, no memory access, and the flags are untouched.

Things to know. lea does not read memory, so the address it computes does not need to be valid. lea does not set flags, which makes it useful in the middle of a chain of conditional code where you can't afford to clobber them. The simple forms ([base + index*scale] or [base + disp]) take one cycle of latency on most recent Intel and AMD cores, with two or more per cycle of throughput[13]. The three-component form ([base + index*scale + disp], any scale) is the "slow LEA" on Sandy Bridge through Skylake-derived cores: port 1 only, three-cycle latency[12]. Ice Lake made it a 1-cycle operation; on Alder Lake's Golden Cove P-cores any LEA with a scaled index takes 2 cycles, and on Zen 3 through Zen 5 the three-component form splits into 2 μops[13]. When tuning for Skylake-class cores, compilers split a three-component LEA into a two-component lea plus an add when it sits on the critical path.

[Back to x86-64 Assembly from Scratch](index.md)
