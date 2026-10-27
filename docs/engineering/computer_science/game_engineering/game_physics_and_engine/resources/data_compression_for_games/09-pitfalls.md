# Pitfalls

## Compression slowed the load

On a fast disk with a slow, high-ratio codec, the decoder is the bottleneck. Choose for decode speed as well as size.

## Compressing already-compressed data

Re-compressing JPEG, encrypted, random, or already-compressed data wastes CPU and usually produces a ratio close to 1:1.

## Using zstd for sampled textures

The GPU cannot sample a general-purpose stream. Use BCn or ASTC for runtime textures, then optionally apply lossless package compression around those texture blobs.

## Ranking codecs by ratio alone

Ratio and decode speed are different axes. LZ4 often wins for load time while losing on package size.

## Confusing zlib, DEFLATE, and gzip

DEFLATE is the compressed stream format. zlib and gzip are container wrappers with headers and checksums.

## One monolithic compressed blob

Without seekable framing, random access requires decoding from the beginning. Compress independent chunks instead.

## Assuming texture ratio depends on the image

In VRAM, BCn and ASTC size is fixed by format. Image content affects quality, not size. On disk, a subsequent LZ pass can still produce content-dependent sizes.
