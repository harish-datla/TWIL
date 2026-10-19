# Calling Conventions: System V vs Windows x64


A hand-written routine that uses rsi as scratch works on every Linux build and corrupts a caller's state on Windows. Both platforms agreed on the instructions and disagreed on the rules for calling functions. Function calls across a public interface on x86-64 follow one of two main specifications: the System V AMD64 ABI[6] (Linux, macOS, the BSDs, PlayStation 4, PlayStation 5) or the Microsoft x64 ABI[7] (Windows, Xbox One, Xbox Series X/S). They disagree on which registers carry arguments, which are callee-saved, how struct returns work, and how the stack is aligned at the call site:
System V AMD64
Linux · macOS · BSD · PS4/PS5
Int args (in order)	rdi, rsi, rdx, rcx, r8, r9
Float args	xmm0–xmm7
Int return	rax (high half: rdx)
Float return	xmm0 (high half: xmm1)
Caller-saved	rax, rcx, rdx, rsi, rdi, r8–r11, all xmm*
Callee-saved	rbx, rbp, r12–r15, rsp
Stack alignment at call	16-byte (so rsp = 16k − 8 on entry)
Red zone	128 bytes below rsp usable without adjusting
Shadow space	None
Microsoft x64
Windows · Xbox (MSVC toolchain)
Int args (in order)	rcx, rdx, r8, r9
Float args	xmm0–xmm3
Int return	rax
Float return	xmm0
Caller-saved	rax, rcx, rdx, r8–r11, xmm0–xmm5
Callee-saved	rbx, rbp, rdi, rsi, rsp, r12–r15, xmm6–xmm15
Stack alignment at call	16-byte (same)
Red zone	None
Shadow space	32 bytes the caller reserves just above the return address

Two differences cause many porting bugs. First, rdi and rsi are callee-saved on Windows but caller-saved (argument registers) on System V. Code that hand-codes assembly without restoring rdi and rsi works on Linux and silently corrupts state on Windows[7]. Second, Windows requires the caller to reserve 32 bytes of "shadow space" just above the return address, where the callee may spill its four register arguments. Code built for one convention that calls code built for the other needs a thunk that translates between them; GCC and Clang can generate one from an ms_abi or sysv_abi function attribute.

Worked example

Call f(int a, float b, int c).

System V counts integer and float arguments separately: a goes in edi (the first integer register), b in xmm0 (the first float register), c in esi (the second integer register).

Windows assigns by position, so each argument gets its slot's register: a in ecx (slot 1), b in xmm1 (slot 2), c in r8d (slot 3). The same call puts c in two different registers.

The widget below shows the same function call under both ABIs side by side. Pick a call; each column shows which register or stack slot receives each argument, with a note on why:
Live · ABI side-by-side
render(pos, scale, flags)
memcpy(dst, src, n)
lerp(a, b, t)
Pick a call with the buttons; each column lists where every argument and the return value go under that ABI, with a note on why. A vec3 argument is more interesting than it looks. System V classifies a 12-byte struct of three floats as two SSE-class eightbytes and passes it in two XMM registers (xmm0 for x,y; xmm1 for z)[6]. Microsoft x64 passes any aggregate whose size isn't 1, 2, 4, or 8 bytes by reference: the caller copies it to a temporary in its own frame and passes a pointer in rcx[7].
Check Your Understanding Q1 of 1

You write a small assembly routine on Linux that uses rsi as a scratch register and don't save/restore it. You build a Windows port. The same code crashes mysteriously in the middle of a long call chain. Why?
Windows uses a different instruction encoding for rsi.
On Windows, rsi is callee-saved. Your routine clobbered the caller's rsi; when control eventually returned to a function that expected rsi to still hold its value, it read garbage. On Linux rsi is caller-saved (it's the 2nd arg register), so the caller knew not to expect it preserved.
Windows passes arguments on the stack; Linux passes them in registers. rsi isn't an argument register on Windows, so it holds garbage.
0/1 correct

[Back to x86-64 Assembly from Scratch](index.md)
