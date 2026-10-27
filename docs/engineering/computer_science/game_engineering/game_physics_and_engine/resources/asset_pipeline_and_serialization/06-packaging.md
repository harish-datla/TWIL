# Packaging


Your build has a file per asset, and startup spends its time opening them. Thousands of small files mean thousands of open() calls and seeks. A package concatenates cooked assets into a few archives plus an index (offset, size, and ID per entry), which cuts I/O overhead and lets the streamer pull contiguous data.

Unreal's .pak is "an archive file format... to store cooked content" with an index for its file system, and UE5's IoStore splits content (.ucas) from metadata (.utoc)[13]; Unity's AssetBundle is a platform-specific archive grouping assets[14]. Compression layers on here (per-entry or whole-archive) and is its own tutorial; the index plus contiguous layout is what makes the prioritized streaming from the File Streaming tutorial feasible.

Worked example

A package holds 10,000 assets. Give each index entry a 64-bit ID, a 64-bit offset and a 64-bit size: 24 bytes, so the whole index is 240,000 bytes, about 234 KiB, read once at startup. Loading an asset is then a lookup in that table and one read at a known offset in a file that's already open, instead of 10,000 separate opens.

[Back to Asset Pipeline & Serialization](index.md)
