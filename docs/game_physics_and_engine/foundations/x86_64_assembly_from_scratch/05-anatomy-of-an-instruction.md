# Anatomy of an Instruction


A crash dump from a stripped build points at the bytes 48 8B 44 F7 10 and nothing else. Being able to turn those back into an instruction tells you what the code was doing when it died. An x86-64 instruction is between 1 and 15 bytes. Its decoded form is a sequence of optional and mandatory fields, in this order[10]:

    Legacy prefixes (0–4 bytes). Address-size override, operand-size override, segment override, repeat prefix, LOCK.
    REX prefix (0–1 byte). Required to encode the 64-bit operand size (for instructions that don't default to it), registers R8–R15, or the SIL/DIL/BPL/SPL byte registers. Starts with the high nibble 0x4; the low nibble carries four bits W, R, X, B.
    Opcode (1–3 bytes). Selects the instruction; in the "+r" forms, part of the opcode byte also encodes a register.
    ModRM (0–1 byte). For instructions with operands, encodes the addressing mode and one or two register fields.
    SIB (0–1 byte). Scale/Index/Base for the memory addressing modes that need it ([rbx + rcx*4 + 0x10]).
    Displacement (0, 1, or 4 bytes in 64-bit code; 8 only for the special moffs forms of mov). The constant offset in a memory operand.
    Immediate (0, 1, 2, 4, or 8 bytes; 8 only for mov r64, imm64). A literal value baked into the instruction.

Worked example

Decode 48 8B 44 F7 10.

48 is REX with W = 1: 64-bit operand size. 8B is the opcode for mov r64, r/m64.

44 is ModRM 01 000 100: mod = 01 (an 8-bit displacement follows), reg = 000 (rax), r/m = 100 (a SIB byte follows). F7 is SIB 11 110 111: scale 8, index 110 (rsi), base 111 (rdi). 10 is the displacement.

Together: mov rax, [rdi + rsi*8 + 0x10], the third instruction in the widget below.

AVX adds two more prefix families that replace REX and carry a "non-destructive source" field, letting vaddps ymm0, ymm1, ymm2 compute ymm0 = ymm1 + ymm2 without clobbering either source. The VEX prefix (2 or 3 bytes) covers AVX/AVX2; the EVEX prefix (4 bytes) covers AVX-512 and adds the mask register selector and rounding controls.

Click an instruction below to see its bytes broken out. Most production disassemblers can show this view (objdump -d -M intel --show-raw-insn, llvm-mc --show-encoding); seeing it once builds the muscle memory:
Live · Instruction-encoding decoder
mov rax, rbx
mov rax, 0x42
mov rax, [rdi + rsi*8 + 0x10]
add rax, rbx
vaddps ymm0, ymm1, ymm2
lea rax, [rdi + rsi*4]
Click an instruction to see its bytes in hex, each colored by the field it belongs to, with a line per field explaining its bits. Each instruction's encoding is from the Intel SDM Volume 2 instruction reference[10]. Assemblers can pick different, equally valid encodings of the same instruction; for example, add rax, rbx can be encoded with the destination in the ModRM reg field (opcode 03) or in the r/m field (opcode 01). The widget shows the form the GNU assembler emits.


[Back to x86-64 Assembly from Scratch](index.md)
