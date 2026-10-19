# Oodle and Console Hardware

A console SSD that reads 5.5 GB/s needs a decoder that keeps up. RAD’s Oodle suite, including Kraken and related codecs, targets high ratios with high decode speed and is distributed with Unreal Engine. Kraken streams can use slower, denser modes when the decoder has spare capacity relative to the drive.

The PlayStation 5 has a raw SSD rate of 5.5 GB/s and a dedicated hardware decompression unit in its I/O complex. Sony quotes roughly 8–9 GB/s of decompressed output for typical 1.5:1 data and up to 22 GB/s for especially compressible data.

## Worked example

At a 1.5:1 ratio, a 5.5 GB/s drive produces `5.5 × 1.5 = 8.25 GB/s` of decoded output. Using zstd’s rough 1.25 GB/s per modern server core as a yardstick would require more than six cores solely for decompression. Kraken has different characteristics, but the scale explains why dedicated hardware is useful.

The claim that the hardware decoder is “equivalent to 9 Zen 2 cores” should be treated as a soundbite, not a measured equivalence: fixed-function hardware is not a substitute for general-purpose cores.

Oodle Texture is also distinct from Oodle Kraken. It prepares BCn texture data so a following LZ pass compresses better. In one reported example, Kraken alone reached 1.82:1, while Oodle Texture plus Kraken reached 3.16:1.
