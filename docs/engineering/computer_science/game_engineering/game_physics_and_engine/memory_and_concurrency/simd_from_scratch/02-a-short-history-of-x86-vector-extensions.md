# A Short History of x86 Vector Extensions


You build with AVX2 turned on and the game crashes at boot on a player's older PC. Or you open a disassembly and find addps next to vpshufb and need to know what hardware each one needs. Both come down to which generation an instruction belongs to. x86 SIMD is nearly thirty years of extensions, each layered on the previous one, so reading what your compiler emits in 2026 means knowing which generation a given mnemonic comes from and what hardware it requires.
Timeline: from MMX (1997) to AVX10 (2025)

In short: SSE (1999) brought 128-bit registers and SSE2 (2001) added doubles and integers to them, and AMD64 made SSE2 the baseline every x86-64 compiler assumes[3]. SSSE3 (2006) added pshufb, the byte shuffle behind many string and lookup tricks[4]. AVX (2011) widened the registers to 256 bits with a three-operand encoding[2], and FMA3 added fused multiply-add[5]. AVX-512 (2016 on) doubled the width again and added mask registers[7], but Intel left it off its hybrid client chips from 2021 to 2024, while AMD added it with Zen 4 and Zen 5[6][37].

The practical state in 2026: a portable x86-64 binary can assume SSE2. -march=x86-64-v3 (the x86-64 psABI level that adds AVX2, FMA, BMI1/2, and a few others) matches the PS5 / Xbox Series feature set and is the natural PC target for code shared with those builds. AVX-512 stays opt-in on x86 client and is standard on current server parts. Arm NEON is the parallel story for Switch, Switch 2, every modern Android phone, Apple Silicon, and Windows on Arm; §15 is the bridge.


[Back to SIMD from Scratch](index.md)
