# Hardware watchpoints: catching the writer

A byte that should always be 0xAB turns into 0xCC at some point in the frame. Who's writing it? Reasoning from the source runs out of road: the culprit could be any function, any thread, any memcpy with a wrong length. A breakpoint stops execution at an instruction; a watchpoint stops it when a specific memory location is read or written, whichever instruction did it. It's one of the most useful debugger features and one of the most underused. The CPU does the checking: x86 has four debug-address registers DR0–DR3, each holding a linear address, with the read/write/execute mask and the access size encoded in DR7[19]. ARMv8 has a similar mechanism through the DBGWVR/DBGWCR register pairs; the architecture allows between 2 and 16 watchpoint pairs and most A-profile cores ship with 4[20].

A watchpoint on that byte traps the single store that broke the invariant and stops with the stack of the thread that did it. x86 data breakpoints trap after the store completes, so the reported PC is the instruction just past the write. The source line the debugger shows is usually the one that wrote, or the next one when the store was the last instruction of its line.

Setting one is one command:
gdb · watch a single byte

# Trap any write to the byte at 0x7ffff7e0a420.
(gdb) watch *(uint8_t *)0x7ffff7e0a420

# Or: trap any write to the cookie field of a known object.
(gdb) watch player->cookie

# rwatch / awatch trap reads / any access. Same DR0-DR3 hardware.
(gdb) rwatch *(uint32_t *)&config->magic
(gdb) awatch *(uint32_t *)&config->magic

Worked example

A 16-byte buffer runs from 0x1010 to 0x101f, and the next object starts at 0x1020. A memcpy told to copy 17 bytes writes dst[0] through dst[16], and dst[16] lands at 0x1010 + 16 = 0x1020. A watchpoint on 0x1020 lets the 16 legitimate writes run and traps the 17th, with the memcpy and its caller on the stack. Without it, you find out frames later, when the next object reads its first byte and gets garbage.

The widget below runs that program. Pretend you don't know which write corrupts the byte just past the buffer (0x1020, the first cell of the third row). Click a cell to set a hardware watchpoint and run; the simulator pauses on the offending write and shows the stack at that instant:
Live · Hardware watchpoint simulator
Run ▶
Step ▶
Reset ⟲
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
CC
live watched caught write
log program loaded. memory pre-filled with 0xCC. 0x1010-0x101f is a 16-byte buffer; 0x1020 is the first byte of the next object.
log click cells to set watchpoints (max 4), then Run.
Click a cell to set or clear a hardware watchpoint, then Run. Each cell is one byte, from 0x1000 at the top left to 0x103f at the bottom right. The simulated program does a memcpy with a slightly wrong length and a memset later in the frame. The watched cell traps the offending write; the old and new values and the stack at the moment of the trap are what your debugger would print. The four-watchpoint limit is the same one you would hit with DR0–DR3 on real hardware.

Two practical limits. First, hardware watchpoints are scarce: four on x86, four on most ARM cores, sometimes up to sixteen. When GDB can't use the hardware for a watch (the expression is too wide, or hardware watchpoints are disabled), it falls back to a software watchpoint: single-stepping plus a value check at every step. The only hint is that it prints Watchpoint 2 instead of Hardware watchpoint 2. Set more hardware watchpoints than the CPU has and GDB reports Could not insert watchpoint when you resume[16]. Software watchpoints are correct but, in GDB's own words, hundreds of times slower than normal execution; a scene that loads in seconds takes minutes. Pick the watch range carefully. Second, x86 hardware watchpoints cover 1, 2, 4, or 8 aligned bytes each (Intel's encoding[19]). Four 8-byte watchpoints cover 32 bytes, so watching a 64-byte struct means a software watchpoint or picking the fields the corruption is known to hit.

