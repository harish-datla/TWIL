# The Core Instructions, by Frequency


A fifty-line function dump looks dense, but most of it is the same dozen instructions over and over. Recent x86-64 has on the order of a thousand distinct mnemonics, and thousands of variants once operand sizes and addressing modes are counted[13]. The fifteen or so in the table below account for most of the scalar code a compiler emits. Sorted roughly by how often they appear:
Mnemonic	What it does	Form you'll usually see
mov	Copy bits. Register-to-register, register-to-memory, memory-to-register, immediate-to-register. The basic data movement instruction.	mov rax, [rdi + 8]
lea	Compute an address (or any 3-operand arithmetic that fits the addressing modes), don't dereference. See §7.	lea rax, [rdi + rsi*4]
add / sub	Integer add/subtract. Two-operand: dst = dst op src.	add rsp, 0x28
imul	Signed multiply. Most often seen as two-operand (imul dst, src → dst *= src) or three-operand with an immediate (imul dst, src, imm → dst = src × imm). These forms keep only the low half of the product, which is the same for signed and unsigned inputs, so compilers use them for unsigned multiplies too.	imul rax, rcx
shl / shr / sar	Bit shift left, logical right, arithmetic right. sar preserves the sign bit; shr doesn't.	shl rax, 3
and / or / xor	Bitwise ops. xor reg, reg is the canonical zeroing idiom.	xor eax, eax
cmp / test	Set the flags register. cmp a, b = subtract without storing; test a, b = AND without storing.	cmp rax, 0
jcc	Conditional jump. je = jump if equal (ZF=1), jne, jl, jg, jb, ja (signed vs unsigned). Follows a cmp or test.	jne loop
jmp	Unconditional jump.	jmp .L7
call / ret	Function call and return. call pushes the return address and jumps; ret pops and jumps.	call malloc
push / pop	Decrement RSP by 8 and store; or load and increment by 8. Used in function prologues/epilogues for callee-saved registers.	push rbp
cmovcc	Conditional move. cmovge dst, src = if SF=OF then dst=src. Branchless conditionals; see §11.	cmovl rax, rdi
setcc	Set a byte to 1 if a condition holds, else 0. Often used with movzx to materialize a 0/1 in a register.	setl al
movzx / movsx	Move with zero-extension or sign-extension. Bridge between 8/16-bit and 32/64-bit registers.	movzx eax, byte ptr [rdi]

The cc suffix on jumps, conditional moves, and setcc is one of sixteen condition codes that test the flags register set by the most recent cmp, test, or arithmetic instruction: z/e (zero/equal), nz/ne, l/g (signed less/greater), b/a (unsigned below/above), s (negative), o (overflow), and the rest. A jcc reads the flags and conditionally jumps; a cmovcc reads the flags and conditionally moves; a setcc reads the flags and conditionally writes 1 to a byte[10].

Worked example

cmp edi, esi with edi = 3 and esi = 0xFFFFFFFF, then a conditional jump.

Read as signed, esi is −1, and 3 < −1 is false, so jl is not taken. Read as unsigned, esi is 4,294,967,295, and 3 < 4,294,967,295 is true, so jb is taken. Same bits, same cmp; the compiler picks jl or jb from the C type.

Memory addressing supports one form, used by mov, lea, and most others: [base + index*scale + displacement], where base and index are 64-bit registers, scale is 1/2/4/8, and displacement is a signed 8 or 32-bit constant. Any subset can be omitted. RIP-relative addressing ([rip + offset]) is the special case used for position-independent globals on x86-64.

[Back to x86-64 Assembly from Scratch](index.md)
