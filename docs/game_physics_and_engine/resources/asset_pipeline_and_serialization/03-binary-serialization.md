# Binary Serialization


The tools write a cooked header on a PC, and the same bytes load fine there but crash on an ARM phone or read as nonsense on another machine. A cooked format is binary. Three fundamentals decide whether it loads correctly and quickly: byte order, alignment, and what you're allowed to put in it.

    Endianness. Endianness is the order a multi-byte number's bytes are stored in. Pick and document one. Most targets a game cares about are little-endian, which is why formats standardize on it: glTF mandates that "all buffer data... MUST use little endian byte order"[4]. "Little-endian everywhere" is still an over-claim (network byte order is big-endian, and some hardware is too), so the format must declare its order rather than assume.
    Alignment. A type with alignment N must sit at an address that's a multiple of N; the compiler pads between fields to satisfy it. Use fixed-width types (<cstdint>) and lock the layout with static_assert, a compile-time check, on size and alignment.
    POD only. To write a struct straight to disk it must be plain data, no virtual functions, because a vtable pointer (the hidden pointer to a class's table of virtual functions) is a runtime address that means nothing in a file[11].

Worked example

The header in §4 has a 4-byte magic at offset 0, a 4-byte version at 4 and a 2-byte flags at 8. The next field, the 4-byte payloadBytes, needs a multiple of 4, so the compiler inserts 2 bytes of padding and places it at 12. The 8-byte contentHash lands at 16, and the struct is 24 bytes with 8-byte alignment. Byte order matters just as much: version 3 is stored little-endian as the bytes 03 00 00 00, and a big-endian machine that reads them without swapping sees 50,331,648.
A misaligned cast is undefined behavior, not just slow

Casting a byte buffer to a struct only works if the buffer satisfies the struct's alignment. On x86-64 an unaligned scalar load works, typically at little or no cost unless it splits a cache line (the 64-byte unit memory moves in on those CPUs) or a page. On ARM it depends on the core and the instruction: older 32-bit cores and some instructions (multi-register loads, atomics) raise an alignment fault, which Linux and Android deliver as a bus error (SIGBUS). And the language rules forbid it regardless of the CPU: in C, even forming the misaligned pointer is undefined behavior, which the compiler may assume never happens, and in C++ reading through it is[8]. Copy with memcpy (or std::bit_cast from an aligned byte array) when alignment isn't guaranteed. In Rust, use #[repr(C)] for a defined field order you can serialize; the default #[repr(Rust)] layout is unspecified.

The inspector lays out an asset header byte by byte, with padding highlighted. Pick an older file version to see its on-disk layout next to the current struct it migrates into:
Load file version:
v1
v2
v3
force pack(1)
Each colored block is a field on a byte ruler numbered from 0, with padding marked "pad". Click v1, v2 and v3, then tick "force pack(1)". The 16-bit flags field leaves a 2-byte gap so the next 32-bit field stays 4-aligned, and the v3 struct is 24 bytes. A v1 file has no flags, so its payloadBytes sits at offset 8 and 4 pad bytes keep the hash 8-aligned; the loader reads that layout and fills flags with its default. A v2 file already has the v3 layout, so only its version number changes. Toggle "force pack(1)" and the padding vanishes: the struct shrinks to 22 bytes with payloadBytes at offset 10 and the hash at 14, both misaligned: undefined behavior in C++ everywhere, tolerated by x86, a fault on some ARM cores and instructions.

[Back to Asset Pipeline & Serialization](index.md)
