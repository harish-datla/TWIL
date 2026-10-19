# Compression and the GPU Decompression Revolution


Storage is the bottleneck, so the obvious optimization is to send fewer bytes across it: a texture compressed 2:1 spends half as long on the drive, as long as something can decompress it fast enough. Shipping engines compress most of their assets on disk, and the decompressor sits between the raw read and the resource that gets bound to a draw call. The question is which compressor and, more importantly in 2026, which processor runs it.
Two layers of compression

Texture data is unusual: most of it lives in block-compressed formats at runtime, not just on disk. BC1 through BC7 (collectively "BCn") are GPU-native compressed formats: each one packs a 4×4 block of texels into a small fixed-size payload, and the GPU's texture sampler decodes them on the fly during every lookup[21]. There's no engine-visible decompression step; the textures stay in this format from disk to render.

The different BC formats trade off bit rate, channel count, and quality. Picking the right one per texture is one of the biggest levers a content team has on VRAM (the GPU's own memory) and disk footprint:
Format 	Bytes / block 	Bits / pixel 	Channels 	Typical use
BC1	8	4	RGB (or RGB + 1-bit α)	Opaque diffuse / albedo textures where the cheapest format is good enough
BC2	16	8	RGB + 4-bit α	Mostly obsolete; superseded by BC3 / BC7
BC3	16	8	RGB + smooth α	Standard for textures with gradient transparency
BC4	8	4	Single channel	Heightmaps, masks, roughness, AO
BC5	16	8	Two channels	Tangent-space normal maps (X and Y; Z is reconstructed)
BC6H	16	8	HDR RGB (no α)	HDR cubemaps, IBL probes, lightmaps
BC7	16	8	RGB or RGBA	The high-quality default. Significantly fewer artifacts than BC1/BC3, same on-disk size as BC3

All BCn formats compress at a fixed ratio (the block-byte count is the same regardless of content), so the encoder's job is to choose the encoding that best approximates the original 16 texels under that fixed budget. BC7 has 8 encoding modes and picks the best one per block; BC1 has two, selected by the order of the block's two endpoint colors. That's a large part of why BC7 looks much better than BC1 or BC3 on photographic textures, at the same 1 byte per pixel as BC3 and twice BC1's.

Once a texture is in BCn, the bits on disk are still bulky enough to be worth compressing further: BCn is fixed-rate, not entropy-coded, so it leaves redundancy an LZ codec can find.

Worked example

A 4096 × 4096 RGBA texture is 16.8 million pixels at 4 bytes each: 64 MiB. As BC7 (1 byte per pixel) it's 16 MiB, and as BC1 (half a byte per pixel) 8 MiB, and the GPU samples either directly. A full mip chain adds about a third: 21.3 MiB for BC7. A stream codec on top then shrinks the disk copy further; BCn data typically compresses about 1.3 to 1.8× (see the codec table below).

In a pipeline that also uses a rate-distortion re-encoder, a shipped texture passes through a chain of encoders at build time, each aimed at a different reader, and the engine undoes only the last one at load time:
eq. 2 · compression stack raw → BCn → Oodle Texture → stream codec → disk

In words: encode to BCn for the GPU, let a re-encoder pick BCn blocks that compress well, then compress everything once more for the disk. Each stage serves a different consumer. BCn is for the GPU sampler (which never sees decompressed data). Oodle Texture[22] is a re-encoder rather than a compressor: it picks BCn blocks the next codec compresses well. RAD quotes about 10% smaller compressed output for near-lossless settings and 20-50% for settings with a small visual difference. The stream codec (GDeflate, Zstd, Oodle Kraken) compresses everything one last time for the disk.
The CPU decompression bottleneck

On PC, until DirectStorage 1.1 (Microsoft's Windows API for fast game loading) arrived in late 2022, decompression happened on the CPU. Read the compressed bytes from disk, decompress on a worker thread, upload to the GPU. One CPU core decodes zlib at roughly 0.4 GB/s and Zstd at roughly 1.5 GB/s[42]; Oodle Kraken, which RAD rates at 3-5× zlib's decode speed[24], lands in the same range, and the speed-tuned LZ-family codecs (dictionary coders such as LZ4 and Selkie, which replace repeated byte runs with back-references) go a few times faster by giving up ratio. Once the SSD can deliver 7 GB/s of compressed data, keeping up takes several cores. NVIDIA's GDeflate writeup measured the result on a system whose uncompressed streaming topped out near 3 GB/s (the PCIe Gen 3 limit): with decompression on the CPU, the CPU became the overall bottleneck and effective throughput fell below what uncompressed streaming delivered[23].
Moving decompression onto the GPU

The PC answer is to do the decompression on the GPU as a compute shader, a general-purpose program the GPU runs across thousands of threads, with a codec designed for SIMD throughput (one instruction working on many values at once). NVIDIA's GDeflate[23] is the best-known example. It's a variant of DEFLATE, the format inside zlib and gzip, that splits the input into 64 KiB tiles, each compressed independently, with the bitstream specifically formatted to expose SIMD-level parallelism. The open spec describes a 32-way sub-stream swizzle, which deals each tile's bits out into 32 interleaved streams, so a warp (a group of 32 GPU threads running in lockstep) can parse it in parallel[40]. GPU vendors can replace the generic shader with tuned driver implementations through metacommands[10]. DirectStorage 1.4[12] adds Zstandard, keeping Zstd's standard format and relying on content split into small independent chunks (details below).

Worked example

The widget's defaults: a 7 GB/s drive reading data compressed 2:1 offers 7 × 2 = 14 GB/s of decompressed output. One core of zlib outputs 0.4 GB/s, so the engine sees 0.4 GB/s, under 3% of what the drive offers, and a faster drive changes nothing. Zstd on one core gets 1.5 GB/s. GDeflate on the GPU reaches about 14 GB/s, and the drive is the limit again.

The throughput picture, when everything is set up correctly:
Live · CPU vs GPU Decompression
CPU zlib
CPU Zstd
CPU Kraken
CPU Mermaid
CPU Selkie
GPU GDeflate
PS5 silicon
compression ratio 2.0×
storage (GB/s raw) 7.0
Run ▶
Reset ⟲
effective throughput
0.4 GB/s
bottleneck
decompressor
The three bars are the drive's raw read rate, the decompressor's output ceiling, and what the engine actually gets. Pick a decoder, press Run, then move the storage slider. At the defaults, CPU zlib reads "0.4 GB/s" with the decompressor as the bottleneck, and GPU GDeflate reads "14.0 GB/s" with storage as the bottleneck. Effective throughput is min(storage × ratio, decompressor): the drive feeds compressed bytes worth storage × ratio of decompressed output per second, and the decompressor's output rate caps it. When the decompressor is the bottleneck, faster storage doesn't help. GPU decompression and the console decoders raise that ceiling above what the drive can feed them. The PS5 ceiling is the hardware block's 22 GB/s peak output; typical content lands at 8-9 GB/s because the 5.5 GB/s drive is the limit.
How the codecs compare

The widget toggle is a teaser; the full picture is a tradeoff between three numbers. Faster decode means the engine is decompressor-limited later or never. Higher ratio means less disk and less bandwidth. Encoder speed only matters at build time but is what determines whether you can run the codec on every CI build or only on a nightly. Approximate published numbers, normalized to a single modern x86 core except where noted:
Codec 	Decode (GB/s) 	Ratio vs raw 	Where it runs 	Notes
zlib (Deflate, lvl 6)	~0.4	~2.0×	1 CPU core	The 1990s baseline. Still the format inside .pak/.zip, but slow.
LZ4 (fast)	~4.0	~2.1×	1 CPU core	Decode-fast, ratio-mediocre. Good when you'd otherwise leave data uncompressed.
Zstandard (lvl 9)	~1.5	~2.5×	1 CPU core	Modern default. Beats zlib on both axes. Added to DirectStorage 1.4.
Oodle Selkie	~5.0	~2.0×	1 CPU core	Tuned for raw decode speed. Pair with content where ratio matters less than latency.
Oodle Mermaid	~3.0	~2.3×	1 CPU core	Middle of the Oodle line, between Selkie's speed and Kraken's ratio.
Oodle Kraken	~1.8	~2.7×	1 CPU core or PS5 silicon	The high-ratio variant. PS5's hardware block typically outputs 8-9 GB/s from the 5.5 GB/s drive, up to 22 GB/s on very compressible data.
Oodle Leviathan	~1.0	~3.0×	1 CPU core	Maximum ratio. Slowest of the line. Useful for cold patches and downloads.
GDeflate (GPU)	~14+	~1.9×	GPU compute shader	Whole GPU, not one core, and in practice limited by the drive feeding it. Slight ratio penalty vs vanilla DEFLATE. PC fast path.
BCPack (Xbox silicon)	~4.8 effective	varies	Xbox Series I/O block	Texture-format-aware: works on BCn directly, so ratio depends on texture content.

Numbers are order-of-magnitude. The zlib, LZ4 and Zstd decode rates follow Zstd's README benchmark (Silesia corpus, one core of a Core i7-9700K)[42]; Oodle's published comparison[41] gives figures for its own codecs on specific hardware. Real ratios depend heavily on what's being compressed: text and code see ~3-4×, BCn texture data sees ~1.3-1.8×, already-compressed audio sees almost nothing.

Each row is a point on a speed-versus-ratio tradeoff curve. The Oodle family spreads four codecs along that curve so a project can match the codec to the asset class: Selkie or Mermaid where decode speed matters most, Leviathan where size matters most (downloads, patches), Kraken as the middle default. Zstd covers a similar range through its compression levels; zlib sits inside the curve, since Zstd beats its ratio with several times its decode speed[42].
Doesn't the PS5 already do this in silicon?

It does, which is why PS5 games have no use for a GPU GDeflate decoder. The hardware Kraken block on the I/O complex typically delivers 8-9 GB/s of decompressed data without spending a shader cycle or a CPU core. GPU compute decompression exists for PC, which has no decompression silicon common to its whole hardware base but does have a GPU that can spare some compute time. On PS5 (and Xbox Series, with its BCPack and LZ decompression blocks), the right answer is to let the silicon do it. The two designs coexist because a shader can gain a new codec with an SDK or driver update, while fixed-function silicon needs a new chip.

The PS5 chose silicon

Sony picked a different path. Instead of using compute shaders, they put a dedicated hardware decompression block on the I/O complex[7]. That block decodes Oodle Kraken (a high-ratio LZ-family codec from RAD Game Tools / Epic)[24] at line rate: the SSD reads 5.5 GB/s of compressed data, the decompressor typically outputs 8-9 GB/s (up to 22 GB/s on data that compresses particularly well), and neither the CPU nor the GPU spends time decoding it. Charles Bloom's post on Oodle Texture for PS5[25] shows why the layered stack matters: on one texture set from a shipped game, Kraken alone compressed 1.82:1 and Oodle Texture plus Kraken 3.16:1. At 5.5 GB/s raw, that ratio would put texture data at about 17 GB/s, though Bloom expects the average across a whole game to land closer to 2:1.

Xbox Series applies the same idea at lower throughput: BCPack (a texture-specific codec) plus a general-purpose LZ decoder, both in silicon, turning 2.4 GB/s of raw NVMe into 4.8 GB/s effective at a 2:1 ratio[8].
Which numbers actually matter

"Effective bandwidth" is what the engine sees: bytes available per second after decompression. It depends on three things: storage raw bandwidth, decompression throughput, and compression ratio. Hitting the platform's advertised number requires all three to be in balance. The console architectures balance them in silicon; PC needs DirectStorage plus a competent compute-shader decoder to get there. Closing that gap is one of DirectStorage's two jobs; the other is cutting the CPU cost of issuing many small reads.
Check Your Understanding Q1 of 2

Your engine on PC gets 1.5 GB/s of effective texture loading from a 7 GB/s NVMe drive. One CPU core is pinned at 100%. The drive is reading about 0.75 GB/s, roughly a tenth of what it can. What is the bottleneck?
The drive: a tenth of its rated bandwidth is what it can actually sustain on this workload once controller and thermal limits kick in.
CPU decompression. The decoder on that one core tops out around 1.5 GB/s of output, so the engine asks the drive for only as much compressed data (0.75 GB/s at 2:1) as the decoder can consume. Moving decompression to the GPU (GDeflate or Zstd via DirectStorage) lifts that ceiling.
PCIe bandwidth: the compressed and decompressed copies of each asset both cross the bus, doubling the traffic.
0/2 correct

[Back to File Streaming for Game Engines](index.md)
