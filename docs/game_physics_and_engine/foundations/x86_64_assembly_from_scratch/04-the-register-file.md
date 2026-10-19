# The Register File


The debugger shows rdx = 0x1122334455667722 after a line that should have put 34 in it. The register names and their partial-write rules explain it. x86-64 exposes sixteen general-purpose 64-bit registers and sixteen (with AVX-512: thirty-two) vector registers. Knowing the names is half of reading a disassembly:
64-bit	32-bit	16-bit	8-bit low	Conventional role (System V)
rax	eax	ax	al	return value (int); scratch
rbx	ebx	bx	bl	callee-saved
rcx	ecx	cx	cl	4th integer arg
rdx	edx	dx	dl	3rd integer arg; high half of 128-bit return
rsi	esi	si	sil	2nd integer arg
rdi	edi	di	dil	1st integer arg
rbp	ebp	bp	bpl	frame pointer; callee-saved
rsp	esp	sp	spl	stack pointer
r8	r8d	r8w	r8b	5th integer arg
r9	r9d	r9w	r9b	6th integer arg
r10–r11	…	scratch
r12–r15	…	callee-saved

Writes to a 32-bit register zero-extend into the 64-bit register. Writes to an 8-bit or 16-bit register do not: they leave the upper bits unchanged. This is an AMD64 architecture rule[8], not a compiler convention, and it is why compilers emit xor eax, eax to zero the full rax: the 32-bit write zero-extends, and the 2-byte encoding is shorter than mov eax, 0 (5 bytes) or mov rax, 0 (7 bytes).

Worked example

rdx holds stale 0x1122334455667788 and cl holds 34 (0x22).

mov dl, cl replaces only the low byte: rdx = 0x1122334455667722. That's the debugger's surprise.

movzx edx, cl writes the 32-bit edx, which zero-extends: rdx = 0x0000000000000022, which is 34.

Vector registers come in three sizes that alias each other:

    xmm0–xmm15 (with AVX-512: xmm0–xmm31): 128 bits. SSE/SSE2/SSE3/SSE4 baseline.
    ymm0–ymm15 (with AVX-512: ymm0–ymm31): 256 bits. AVX/AVX2. The lower 128 bits of ymm0 is xmm0.
    zmm0–zmm31: 512 bits. AVX-512. The lower 256 bits is ymm.

The aliasing is not free. Running legacy-encoded SSE instructions while the upper halves of the YMM registers hold data from VEX-encoded AVX code costs extra on Intel cores: on Sandy Bridge through Broadwell each switch triggers a state save or restore of dozens of cycles, and on Skylake and later each legacy SSE instruction instead carries an extra blend μop and a dependency on the full register[3]. The rule the compiler follows hasn't changed: emit vzeroupper before leaving AVX code. Hand-written assembly that mixes the two has to do the same.

Step through eight instructions and watch the register file update. Each row is one 64-bit register, shown as its high and low 32 bits in hex. The registers start out holding leftover values from earlier code, so the partial-write rules are visible. The arrow marks the next instruction; orange highlights the register just written:
Live · Register-file stepper
Step ▶
Run all ▶▶
Reset ⟲
Each row is one 64-bit register, its high and low halves in hex; the panel lists the program with a marker on the next instruction. Press Step to run one instruction at a time and watch which bits each write changes, or Run all. A small program: add the two integer arguments (edi = 7, esi = 10), double the sum with lea, and return (a + b) * 2 = 34 in eax. The 32-bit writes (mov eax, edi, lea ecx, …) clear bits 63..32 of the full register. The 8-bit write mov dl, cl changes only the low byte of rdx and leaves its other 56 bits holding stale data; movzx edx, cl is the clean way to widen a byte.
Check Your Understanding Q1 of 1

A compiler emits xor eax, eax at the top of a function. Why this, instead of mov rax, 0?
The XOR is two bytes; the MOV is seven. Smaller code, and the CPU recognizes xor reg, reg as a zeroing idiom that doesn't depend on the previous value of rax.
mov rax, 0 doesn't exist in x86-64; the assembler would reject it.
XOR is a faster instruction than MOV.
0/1 correct

[Back to x86-64 Assembly from Scratch](index.md)
