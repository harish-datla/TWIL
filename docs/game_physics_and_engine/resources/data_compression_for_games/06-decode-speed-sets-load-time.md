# Decode Speed Sets Load Time

On a fast NVMe drive, the disk may finish reading the compressed package almost instantly while one CPU core remains busy decompressing. Loading compressed data is a two-stage pipeline: the disk delivers compressed bytes and the decoder reconstructs the original. When both overlap, the slower stage sets the pace.

## Load-time model

```text
load time ≈ max(
    size ÷ (ratio × disk speed),
    size ÷ (cores × decode speed)
)
```

The first term is disk time; the second is decoder time. Better ratio shrinks only the disk term. A faster decoder or more cores shrinks the second.

## Worked example

Load 1 GB from a 5 GB/s NVMe drive. With zstd at 3.05:1, the disk read takes about `1 ÷ (3.05 × 5) = 0.07 s`, but one core needs about 0.80 s to decode. The compressed load takes 0.80 s—four times slower than reading the raw data in 0.2 s. Four independent chunks decoded on four cores reduce the decode time to about 0.20 s.

On the 0.5 GB/s SATA drive from [Why Compress](01-why-compress.md), the same compression is beneficial. Whether compression helps depends on the drive, codec, and available cores.

Two API shapes matter: block APIs process a whole buffer with a known output size; streaming APIs use bounded memory while data arrives. Random access into a large compressed asset needs seekable framing: independent compressed blocks that can be decoded separately. A monolithic stream forces decoding from its beginning.
