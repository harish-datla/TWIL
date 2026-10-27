# A Short History of the Architecture You're Reading


Why does a 64-bit instruction start with a prefix byte, and why does one register have four names (rax, eax, ax, al)? Because x86-64 carries nearly fifty years of architectural decisions, each made with the previous decade of code in mind. The instruction encoding has prefixes whose purpose is to extend an earlier 16-bit encoding. The calling conventions changed when 64-bit mode doubled the general-purpose register count. The vector extensions arrived in four named generations. Much of this reads as arbitrary until you see what each piece replaced.
Timeline: the 8086 to APX

The consequence for what you read in 2026: most disassembly is plain x86-64 with SSE2 floating point. Hot paths use AVX2 (256-bit) on PC and on PS5 / Xbox Series. AVX-512 shows up in compression, simulation, and ML kernels, and rarely in shipped game code because of the Intel client-side gap from Alder Lake through Arrow Lake. Arm, touched on briefly in §12, is the other architecture worth knowing: Nintendo Switch and Switch 2, Apple Silicon, and a growing fraction of Windows-on-Arm devices all ship it, and the calling conventions, register file, and SIMD model (NEON, SVE2) follow different rules.


[Back to x86-64 Assembly from Scratch](index.md)
