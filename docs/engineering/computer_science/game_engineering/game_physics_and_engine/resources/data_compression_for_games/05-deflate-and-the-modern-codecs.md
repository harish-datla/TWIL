# DEFLATE and the Modern Codecs

Codec benchmarks rank differently depending on whether you sort by size or speed. Most codecs here use LZ77 followed by entropy coding; LZ4 skips the entropy stage for speed.

- **LZ4:** Very fast and lower ratio; decoding can approach memory-bandwidth limits.
- **zstd:** Tunable from fast negative levels through level 22, using FSE (a table-driven ANS variant) plus Huffman coding for literals.
- **Brotli:** Competitive ratios, but slower decoding and a static dictionary tuned especially for web text.
- **DEFLATE:** The classic LZ77 plus Huffman combination, with a 32 KiB window. It is the stream used by zlib, gzip, and PNG; zlib and gzip are container wrappers.

## Calling zstd

```cpp
std::vector<char> dst(ZSTD_compressBound(src.size()));
size_t n = ZSTD_compress(dst.data(), dst.size(),
                         src.data(), src.size(), 3); // level 3
if (ZSTD_isError(n)) { /* handle */ }
dst.resize(n);
// Decode with the size from the frame header and ZSTD_decompress.
```

## Worked comparison

For 1 GB of Silesia-like data, representative one-core measurements are approximately:

| Codec | Stored size | Decode time |
| --- | ---: | ---: |
| LZ4 | 476 MB | 0.27 s |
| zstd level 3 | 328 MB | 0.80 s |
| DEFLATE / zlib level 6 | 322 MB | 2.9 s |
| Brotli level 5 | 281 MB | 2.2 s |

zstd level 3 is close to DEFLATE’s size at about a quarter of its decode time. LZ4 gives up roughly 150 MB to decode about three times faster than zstd. Increasing zstd from level 1 to 22 improves ratio, but most extra encoding cost occurs on the packaging side. On already-compressed input, all ratios collapse toward 1:1.
