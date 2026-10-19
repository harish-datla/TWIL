# Bundles: Why One Big File Beats a Million Little Ones


Your build ships one file per asset, tens of thousands of them, and startup spends hundreds of milliseconds opening files before it reads a byte. After how to read fast comes what to read, and in what shape. The naïve approach is one file per asset: one texture per .png, one mesh per .fbx. Shipping builds rarely stay that way. They bundle thousands of assets into a few large files, each with a manifest (an index of names, offsets and sizes). Unreal calls them .pak files; Unity calls them AssetBundles[26]; the WAD files of 1993's Doom are an early, widely known example.
Why bundling helps

    Fewer opens. Opening a file costs tens of microseconds or more (path lookup, permission checks, antivirus filters on Windows). Opening 10,000 files costs hundreds of milliseconds. One open + 10,000 reads at known offsets is much faster.
    Layout control. You can place co-loaded assets next to each other on disk, so a 64 KB read pulls in the whole group instead of needing one seek per asset.
    Filesystem overhead. Every file carries metadata (an inode or MFT record, a directory entry) and rounds up to a whole allocation cluster. 10,000 tiny files cost more space and more metadata reads than one large file plus a manifest.
    Streaming-friendly format. The bundle's internal layout can be aligned to compression-block boundaries, sector boundaries, or anything else the streamer cares about.

Worked example

A market stall needs 40 textures of 64 KB each, 2.5 MiB in all, from an HDD (about 10 ms per seek, 150 MB/s sequential). As 40 scattered files that's 40 seeks plus their transfers: about 417 ms. Packed next to each other in a bundle it's one seek plus 2.5 MiB of sequential reading: 10 + 17 ≈ 27 ms, about 15 times faster for the same bytes.
bundle.h · a minimal bundle format

// File layout:
//   [Header]          fixed-size, names the version + the manifest offset.
//   [Asset bytes]     concatenated, possibly compressed, aligned to 4 KB.
//   [Manifest entries] one per asset: id, offset, compressed size, uncompressed size, flags.
// The manifest lives at the END so writers can stream assets without seeking back.
#include <cstdint>   // uint32_t / uint64_t

struct BundleHeader {
  char     magic[4];          // "MPGB"
  uint32_t version;
  uint64_t manifestOffset;
  uint32_t manifestEntryCount;
  uint32_t defaultCodec;        // 0 = none, 1 = zstd, 2 = gdeflate, ...
};

struct ManifestEntry {
  uint64_t assetId;             // hash of the logical name
  uint64_t byteOffset;          // where the asset starts in the bundle
  uint32_t compressedSize;
  uint32_t uncompressedSize;
  uint32_t flags;               // codec override, alignment hints, etc.
  uint32_t reserved;
};

Patching is the catch

Single huge bundles are a problem when a patch changes one byte of one asset inside a 20 GB file. The patcher either re-downloads the bundle (bad), uses binary diffing (better, fragile), or splits the bundle into smaller "chunks" it can replace independently (the common modern answer). Steam's content system splits every file into roughly 1 MB chunks and reuses the unchanged ones when a new build is uploaded[49]; Unreal's IO Store splits .pak into .utoc/.ucas with chunked content.

[Back to File Streaming for Game Engines](index.md)
