# Packing: Color and Bitfields


A particle system stores a color per particle. Four separate bytes per color work, but it's often handier to keep the color in one 32-bit word, red, green, blue and alpha side by side, so it copies and compares as a single integer. Shifts and masks are how you stuff several small values into one word and pull them back out.

For an 8-bit-per-channel RGBA color, packing shifts each channel into its own byte of the word and ORs them together. Unpacking shifts the wanted byte down to the bottom and ANDs with 0xFF to drop everything above it.

Worked example

The color packer below starts at R = 224, G = 122, B = 95, A = 255. In hex those are E0, 7A, 5F and FF.

Pack: (224 << 24) | (122 << 16) | (95 << 8) | 255 = 0xE07A5FFF. Each channel is one pair of hex digits.

Unpack green: shift right by 16 to get 0xE07A, then AND with 0xFF to keep 0x7A = 122.
Pack and unpack RGBA8888
C / C++
Rust

uint32_t pack(uint8_t r, uint8_t g, uint8_t b, uint8_t a) {
    return (uint32_t(r) << 24) | (uint32_t(g) << 16)
         | (uint32_t(b) <<  8) |  uint32_t(a);
}

// Cast to uint32_t BEFORE shifting. (uint8_t(r) << 24) would promote to
// int and, for r >= 128, push a 1 into the sign bit: undefined behavior
// in C and before C++20 (§7).

uint8_t red(uint32_t c)   { return (c >> 24) & 0xFF; }
uint8_t green(uint32_t c) { return (c >> 16) & 0xFF; }
uint8_t blue(uint32_t c)  { return (c >>  8) & 0xFF; }
uint8_t alpha(uint32_t c) { return  c        & 0xFF; }

Squeeze the same color into 16 bits and you get RGB565: 5 bits of red, 6 of green and 5 of blue. It's still used for framebuffers and textures where bandwidth matters more than fidelity[15]. Packing drops the low bits of each channel (r >> 3, g >> 2, b >> 3), and those bits are gone. A smooth 8-bit gradient collapses to 32 (or 64) levels, which is the banding you see in 16-bit gradients. Unpacking has to invent the low bits back, usually by copying the high bits down into the gap: (r5 << 3) | (r5 >> 2). That maps the top code to 255 exactly (0b11111 becomes 0b11111111) and 0 to 0, so black and full intensity survive the round trip. A plain r5 << 3 tops out at 248. Color Cycling §3 applies these conversions to VGA and Amiga palette entries.

Worked example

Pack (224, 122, 95) as RGB565: 224 >> 3 = 28, 122 >> 2 = 30, 95 >> 3 = 11. The word is (28 << 11) | (30 << 5) | 11 = 0xE3CB.

Unpack with replication: red (28 << 3) | (28 >> 2) = 224 | 7 = 231, green (30 << 2) | (30 >> 4) = 120 | 1 = 121, blue (11 << 3) | (11 >> 2) = 88 | 2 = 90.

So (224, 122, 95) comes back as (231, 121, 90), off by 7 + 1 + 5 = 13 in total. Replication made red slightly worse here (224 would have survived a plain << 3); it's chosen because it gets the two ends of the range exactly right.
Live · Color packer
RGBA8888
RGB565
R 224
G 122
B 95
A 255
input
stored

packed
    0xE07A5FFF
bits
    11100000 01111010 01011111 11111111

R 11100000G 01111010B 01011111A 11111111
lossless · 32 bits · 8 per channel
The sliders set an 8-bit color; the left swatch is that input and the right swatch is what the format stores and unpacks, with the packed word and its channel layout below. In RGBA8888 the two swatches always match. Switch to RGB565: the stored color drifts from the input as the low bits of each channel are thrown away, and the readout gives the reconstruction and the error, (231, 121, 90) and 13 at the starting color. Slide red slowly in 565 mode and the swatch steps instead of glides: only 32 red levels survive where 8-bit had 256. The error is the sum of the per-channel gaps between input and reconstruction.
Endianness is not bit order

People often assume that packing colors with shifts depends on endianness, the order in which a CPU stores a multi-byte value's bytes in memory. It doesn't. Shifts operate on the value in a register, where bit i always means 2i, the same on every machine. Endianness only describes the order bytes land in memory when you store that register. (r << 24) | ... produces the identical number on a little-endian and a big-endian CPU; the bytes only reshuffle when you write the word to a buffer and read it back as bytes. When you genuinely need to swap byte order (network packets, file formats), that is a separate operation: bswap, or std::byteswap in C++23.

[Back to Bit Shifting from Scratch](index.md)
