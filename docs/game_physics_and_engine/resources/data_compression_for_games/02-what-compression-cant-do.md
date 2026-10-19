# What Compression Can’t Do

Already-compressed files such as MP4, Ogg, and JPEG usually gain almost nothing from another lossless pass. The reason is a hard floor. Shannon’s source-coding theorem says that no lossless coder can, on average over a source, use fewer bits per symbol than the source’s entropy: the average surprise per symbol, measured in bits.

Data where a few values appear most of the time is predictable, so a coder can give common values short codes and rare values long codes. Data where every value is equally likely gives it nothing to exploit. Compressed files look like that because removing predictability is what the first compressor did; good encryption makes its output look random on purpose.

## Worked example

A mask texture stores one byte per pixel: 90% of pixels are 0 and 10% are 255. If pixels are independent, its entropy is:

```text
0.9 × log₂(1 ÷ 0.9) + 0.1 × log₂(1 ÷ 0.1)
= 0.9 × 0.152 + 0.1 × 3.32
= 0.47 bits per pixel
```

The file spends 8 bits per pixel, so a good coder can approach a 17:1 ratio. For encrypted bytes, all 256 values are equally likely, so every byte carries `log₂ 256 = 8` bits—the same 8 bits it already uses.

```text
H = Σ p(x) × log₂(1 ÷ p(x))
```

The bound is an average over the source, not every individual message. Lossy compression can go below it by throwing information away; that is a different problem.

## Why no coder shrinks every input

Run-length encoding (RLE) replaces a repeated-byte run with a `(count, value)` pair. A 64-pixel row containing 60 zeros and four 255 values becomes `(60, 0) (4, 255)`: four bytes instead of 64. Data with no runs expands: `ABCD` becomes `1A1B1C1D`.
