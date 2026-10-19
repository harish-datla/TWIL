# Versioning & Migration


You add a flags field to the header, and every file the artists saved last month fails to load. Data outlives the code that wrote it, so a format must evolve. Two valid strategies:

    Version tag + migration. The header carries a version; the loader switches on it and migrates old data up to the current in-memory form. Total control, but you write every migration.
    Schema evolution. Compatibility is built into the format's structure instead. Protocol Buffers key fields by number, not name: the number "cannot be changed once your message type is in use," old binaries skip fields they don't recognize (and keep them when re-serializing), and a deleted field's number must be reserved so it's never reused[7]. FlatBuffers uses per-table vtables so adding fields at the end and deprecating (never deleting or reordering) keeps forward and backward compatibility[6].

Bumping the version doesn't make old data readable

A version tag only detects which layout you have; you still need a loader for each old layout and a default-fill rule for fields it lacked, or old files fail to load. The version field has to sit at the same offset in every version, so it can be read before the rest of the layout is known. And these are different mechanisms: a version tag plus migration converts old to new; schema evolution makes one layout tolerate added and removed fields without a migration step. Forward compatibility (old code reads new data) and backward compatibility (new code reads old data) are also distinct; state which you're claiming.

Worked example

A v1 file arrives. The loader reads 4 bytes at offset 4, the one place every version keeps its version number, and gets 1. In v1 there was no flags field, so payloadBytes sits at offset 8, then 4 bytes of padding, then the hash at 16: still 24 bytes. The loader copies those bytes into the old struct, then builds the current one field by field, with flags set to its default of 0 and the version rewritten to 3. A v2 file already has the v3 layout, so only its version number changes.
A versioned header, loaded and migrated from any supported version
C++
Rust

#include <cstddef>
#include <cstdint>
#include <cstring>

// The current (v3) header, identical on disk and in memory.
struct AssetHeader {              // fixed-width, no virtuals, safe to write as bytes
    uint32_t magic;               // sentinel: catch a wrong/corrupt file early
    uint32_t version;             // drives the migration switch
    uint16_t flags;               // added in v2; a 2-byte pad follows so the next field is 4-aligned
    uint32_t payloadBytes;
    uint64_t contentHash;         // 8-byte aligned at offset 16
};
static_assert(sizeof(AssetHeader) == 24, "layout drifted");
static_assert(alignof(AssetHeader) == 8, "alignment drifted");

// v1 had no flags field, so payloadBytes sat at offset 8.
struct AssetHeaderV1 {
    uint32_t magic;
    uint32_t version;
    uint32_t payloadBytes;
    uint64_t contentHash;         // 4 pad bytes before it keep it 8-aligned
};
static_assert(sizeof(AssetHeaderV1) == 24, "v1 layout drifted");

// Read whichever header version the file holds into the current struct.
// memcpy rather than a pointer cast: the file buffer need not be aligned.
// The file is little-endian and so is every current PC and console CPU, so
// the bytes are copied as-is; a big-endian host would have to swap them.
bool loadHeader(const std::byte* bytes, size_t size, AssetHeader& out) {
    if (size < 8) return false;
    uint32_t version;              // same offset (4) in every version
    std::memcpy(&version, bytes + offsetof(AssetHeader, version), sizeof version);
    switch (version) {
        case 1: {
            if (size < sizeof(AssetHeaderV1)) return false;
            AssetHeaderV1 old;
            std::memcpy(&old, bytes, sizeof old);
            out = AssetHeader{old.magic, 3, 0, old.payloadBytes, old.contentHash};  // flags: default 0
            return true;
        }
        case 2:                    // same layout as v3; only the number changes
        case 3:
            if (size < sizeof(AssetHeader)) return false;
            std::memcpy(&out, bytes, sizeof out);
            out.version = 3;
            return true;
        default:
            return false;          // unknown or newer: refuse
    }
}


[Back to Asset Pipeline & Serialization](index.md)
