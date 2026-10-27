# Data Compression for Games

Your game is a 60 GB download, and players with fast SSDs still sit on loading screens. Compression can help with both, but only as a trade: it spends CPU time so fewer bytes cross the disk and the network. This series explains what compression can and cannot shrink, how LZ77 and Huffman coding turn repetition into fewer bits, how to choose between zstd, LZ4, and related codecs, and why decoder speed often matters more than ratio on a fast drive. GPU texture compression is a separate tool with different rules.

Time: ~55 min  
Level: Mid  
Prerequisites: Asset Pipeline (packages), File Streaming, and basic bit shifting are helpful but not required.  
Stack: C++ & Rust

## What you’ll be able to do

- Estimate whether compression will speed up or slow down a load from disk speed, ratio, and decoder speed.
- Compute entropy, understand the floor no lossless coder can beat, and identify data that will not shrink.
- Trace how LZ77 back-references and Huffman codes turn repeated, lopsided data into fewer bits.
- Choose between LZ4, zstd levels, DEFLATE, and Brotli for downloads, packages, and load paths.
- Explain why runtime textures use BCn or ASTC instead of zstd, and how both forms of compression stack in a shipped package.

## Sections

- [Why Compress](01-why-compress.md)
- [What Compression Can’t Do](02-what-compression-cant-do.md)
- [LZ77: The Sliding Window](03-lz77-the-sliding-window.md)
- [Entropy Coding](04-entropy-coding.md)
- [DEFLATE and the Modern Codecs](05-deflate-and-the-modern-codecs.md)
- [Decode Speed Sets Load Time](06-decode-speed-sets-load-time.md)
- [Oodle and Console Hardware](07-oodle-and-console-hardware.md)
- [GPU Texture Compression](08-gpu-texture-compression.md)
- [Pitfalls](09-pitfalls.md)
- [What’s Next](10-whats-next.md)
