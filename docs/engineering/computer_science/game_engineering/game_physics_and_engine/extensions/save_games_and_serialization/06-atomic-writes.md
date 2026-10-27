# Atomic writes

The power cuts out while the game is writing the save. A save is overwritten in place at your peril. A crash, a power loss, or a killed process partway through the write leaves the file half old data and half new, and unreadable. That is the worst outcome, because the previous good save is gone too. The fix is an atomic write: write a temp file in the same directory, force it to disk, then rename it over the real save.

Pick a strategy and crash mid-write. In-place corrupts the only save; temp-then-rename leaves the previous save fully intact, because the real file is never touched until the rename:
Strategy:
temp + rename
in-place
Save
Crash now
Reset
Press Save, then Crash now before the bar fills; then switch to in-place, Reset and do the same. temp + rename: the real save.dat is never touched until a full write is renamed over it, so a crash leaves the intact previous save. in-place: the write overwrites the only copy, so a crash mid-write corrupts it. The rename is the atomic swap; the temp file must live in the same directory (filesystem) for it to be atomic.

Worked example

The widget writes a save as 24 blocks over about 1.8 s. Crash in place at block 10 and save.dat holds 10 new blocks and 14 old ones, a file no version of your code ever wrote. With temp + rename, the crash leaves 10 new blocks in save.tmp, which is discarded, and save.dat still holds all 24 blocks of the previous save. The player loses only the progress since the last save.
Atomic write stops torn writes, not logical corruption

Temp-then-rename guarantees you never get a half-written file, a torn write. It does not protect against bytes that go bad after the write (a failing drive, a truncated copy) or against a logically-invalid save, a migration bug that writes complete-but-wrong data. A checksum validated on load catches the damaged bytes. It can't catch the migration bug: the checksum was computed over the wrong data, so it matches. A rolling backup slot (keep the previous save.bak) covers both, because when the newest save fails a check or turns out to be wrong, there's an older one to load. And fsync before the rename is not optional. flush() only moves data to the OS buffer; without the sync, a power loss can leave the save replaced by a file whose bytes never reached the disk[12].
The whole save path

The program below puts §2 through §6 into one file format and one save routine. The file starts with a 16-byte header: a magic number, so the loader can reject a file that isn't a save at all; the format version; the payload length; and a CRC-32 of the payload, the same checksum a gzip file stores for its data[13]. Change any single bit of the payload and the CRC changes, so a damaged file fails the comparison on load. It catches accidents, not cheating: anyone who edits a save on purpose can recompute it.

Saving writes save.tmp, flushes it to the device, renames the old save.dat to save.bak, renames save.tmp to save.dat, and on POSIX flushes the directory too. POSIX requires a rename over an existing file to be atomic: anyone looking sees the old file or the new one, never neither[14]. Linux's fsync documentation warns that flushing a file doesn't guarantee its directory entry reached the disk, which takes a separate fsync on the directory[15].

On Windows, MoveFileEx with MOVEFILE_WRITE_THROUGH doesn't return until the move is on disk[16]. On Apple platforms, fsync hands the data to the drive but doesn't make the drive write it out of its own cache; fcntl(F_FULLFSYNC) does, so the C++ listing uses that there[17]. Rust's File::sync_all already makes the same choices: fsync on Linux, F_FULLFSYNC on Apple, FlushFileBuffers on Windows. Its fs::rename calls rename on Unix and MoveFileExW on Windows, but without the write-through flag, so on Windows the Rust version's rename may not be on disk yet when save returns[18].

Loading checks the header and checksum before trusting a byte, checks the entity count against the payload size before allocating anything, migrates a v1 file by defaulting stamina, re-resolves references by id in a second pass, and falls back to save.bak when save.dat fails.
A versioned, checksummed save with an atomic write, a backup and a validated load (complete program)
C++
Rust

// A save file you can trust: a versioned binary format with a CRC-32 checksum,
// written atomically (temp file, flush to disk, rename) with the previous save
// kept as a backup, then loaded with validation, migration and reference fix-up.
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <iterator>
#include <optional>
#include <string>
#include <unordered_map>
#include <vector>
#ifdef _WIN32
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#else
#include <fcntl.h>
#include <unistd.h>
#endif

using Bytes = std::vector<std::uint8_t>;
constexpr std::uint32_t kMagic = 0x45564153;   // the bytes "SAVE" read as a little-endian u32
constexpr std::uint32_t kVersion = 2;           // v2 added stamina
constexpr std::uint32_t kHeaderSize = 16;       // magic, version, payload length, CRC-32
constexpr std::uint32_t kNoTarget = 0;

struct Entity {
    std::uint32_t id = 0;
    float x = 0, y = 0, health = 0, stamina = 0;
    std::uint32_t targetId = kNoTarget;   // what the file stores: an id, never a pointer
    int target = -1;                      // what the game uses after fix-up: an index, -1 for none
};
using World = std::vector<Entity>;

// CRC-32 as gzip defines it (RFC 1952: reflected polynomial 0xEDB88320), bit by bit.
std::uint32_t crc32(const std::uint8_t* data, std::size_t size) {
    std::uint32_t crc = 0xFFFFFFFFu;
    for (std::size_t i = 0; i < size; ++i) {
        crc ^= data[i];
        for (int bit = 0; bit < 8; ++bit) crc = (crc >> 1) ^ (0xEDB88320u & (0u - (crc & 1u)));
    }
    return ~crc;
}

// Little-endian writes and bounds-checked reads, so the format is the same on every machine.
void putU32(Bytes& out, std::uint32_t value) {
    for (int i = 0; i < 4; ++i) out.push_back(std::uint8_t(value >> (8 * i)));
}
void putF32(Bytes& out, float value) { std::uint32_t bits; std::memcpy(&bits, &value, 4); putU32(out, bits); }
struct Reader {
    const Bytes& bytes; std::size_t at = 0; bool ok = true;
    std::uint32_t u32() {
        if (at + 4 > bytes.size()) { ok = false; return 0; }       // never read past the end
        std::uint32_t value = 0;
        for (int i = 0; i < 4; ++i) value |= std::uint32_t(bytes[at + i]) << (8 * i);
        at += 4;
        return value;
    }
    float f32() { std::uint32_t bits = u32(); float value; std::memcpy(&value, &bits, 4); return value; }
};

Bytes encode(const World& world, std::uint32_t version) {
    Bytes payload;
    putU32(payload, std::uint32_t(world.size()));
    for (const Entity& entity : world) {
        putU32(payload, entity.id);
        putF32(payload, entity.x); putF32(payload, entity.y); putF32(payload, entity.health);
        if (version >= 2) putF32(payload, entity.stamina);
        putU32(payload, entity.targetId);
    }
    Bytes file;
    putU32(file, kMagic); putU32(file, version); putU32(file, std::uint32_t(payload.size()));
    putU32(file, crc32(payload.data(), payload.size()));
    file.insert(file.end(), payload.begin(), payload.end());
    return file;
}

// Validate everything before trusting anything; on failure, say which check failed.
std::optional<World> decode(const Bytes& file, std::string& error) {
    Reader header{file};
    std::uint32_t magic = header.u32(), version = header.u32(), length = header.u32(), storedCrc = header.u32();
    if (!header.ok) { error = "shorter than a header"; return std::nullopt; }
    if (magic != kMagic) { error = "not a save file"; return std::nullopt; }
    if (version == 0 || version > kVersion) {   // a newer build wrote it: refuse rather than misread
        error = "unknown version " + std::to_string(version);
        return std::nullopt;
    }
    if (length != file.size() - kHeaderSize) { error = "length mismatch (truncated?)"; return std::nullopt; }
    if (crc32(file.data() + kHeaderSize, length) != storedCrc) {
        error = "checksum mismatch";
        return std::nullopt;
    }

    Reader in{file, kHeaderSize};
    std::uint32_t count = in.u32();
    std::size_t bytesPerEntity = version >= 2 ? 24 : 20;
    if (std::uint64_t(count) * bytesPerEntity + 4 != length) {   // before allocating; 64-bit math can't wrap
        error = "payload doesn't match its own counts";
        return std::nullopt;
    }
    World world(count);
    for (Entity& entity : world) {                     // pass 1: create every object
        entity.id = in.u32();
        entity.x = in.f32(); entity.y = in.f32(); entity.health = in.f32();
        entity.stamina = version >= 2 ? in.f32() : 100.0f;   // migrate: v1 had no stamina
        entity.targetId = in.u32();
    }
    std::unordered_map<std::uint32_t, int> indexOf;
    for (int i = 0; i < int(world.size()); ++i) indexOf[world[i].id] = i;
    for (Entity& entity : world) {                     // pass 2: re-resolve references by id
        auto found = indexOf.find(entity.targetId);
        entity.target = found == indexOf.end() ? -1 : found->second;   // missing target: drop the reference
    }
    return world;
}

#ifndef _WIN32
// fsync, except on Apple platforms, where fsync can stop at the drive's own write cache.
bool flushToDevice(int fd) {
#ifdef __APPLE__
    return ::fcntl(fd, F_FULLFSYNC) != -1;
#else
    return ::fsync(fd) == 0;
#endif
}
#endif

// Write bytes to a file and force them to the storage device before returning.
bool writeDurably(const std::string& path, const Bytes& bytes) {
#ifdef _WIN32
    HANDLE file = CreateFileA(path.c_str(), GENERIC_WRITE, 0, nullptr, CREATE_ALWAYS,
                              FILE_ATTRIBUTE_NORMAL, nullptr);
    if (file == INVALID_HANDLE_VALUE) return false;
    DWORD written = 0;
    bool ok = WriteFile(file, bytes.data(), DWORD(bytes.size()), &written, nullptr) && written == bytes.size()
              && FlushFileBuffers(file);
    CloseHandle(file);
    return ok;
#else
    int fd = ::open(path.c_str(), O_WRONLY | O_CREAT | O_TRUNC, 0644);
    if (fd < 0) return false;
    std::size_t done = 0;
    while (done < bytes.size()) {                      // write() may write less than asked
        ssize_t chunk = ::write(fd, bytes.data() + done, bytes.size() - done);
        if (chunk <= 0) { ::close(fd); return false; }
        done += std::size_t(chunk);
    }
    bool ok = flushToDevice(fd);                        // to the device, not just the OS cache
    ::close(fd);
    return ok;
#endif
}

// Replace `to` with `from` by renaming, never by copying bytes. On POSIX, readers see the old
// file or the new one, never a mix.
bool replaceFile(const std::string& from, const std::string& to) {
#ifdef _WIN32
    return MoveFileExA(from.c_str(), to.c_str(), MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH);
#else
    return std::rename(from.c_str(), to.c_str()) == 0;
#endif
}

void syncDirectory() {                                 // POSIX: make the renames themselves durable
#ifndef _WIN32
    int fd = ::open(".", O_RDONLY);
    if (fd >= 0) { flushToDevice(fd); ::close(fd); }
#endif
}

bool save(const World& world) {
    if (!writeDurably("save.tmp", encode(world, kVersion))) return false;
    replaceFile("save.dat", "save.bak");              // keep the previous save; fails harmlessly the first time
    bool ok = replaceFile("save.tmp", "save.dat");
    syncDirectory();
    return ok;
}

// Test helper: overwrite a file with no durability, to damage it on purpose.
void overwrite(const std::string& path, const Bytes& bytes) {
    std::ofstream(path, std::ios::binary).write(reinterpret_cast<const char*>(bytes.data()), std::streamsize(bytes.size()));
}

Bytes readFile(const std::string& path) {
    std::ifstream file(path, std::ios::binary);
    return Bytes(std::istreambuf_iterator<char>(file), {});
}

// Try the newest save, then the backup.
std::optional<World> load() {
    for (const char* path : {"save.dat", "save.bak"}) {
        std::string error;
        if (auto world = decode(readFile(path), error)) {
            if (world->size() < 2) return world;            // valid, but no player and guard to print
            const Entity& player = (*world)[0];
            int guardTarget = (*world)[1].target;
            std::printf("  loaded %s: player health %.0f, stamina %.0f; guard's target: %s\n", path,
                        player.health, player.stamina, guardTarget < 0 ? "none" : "the player");
            return world;
        }
        std::printf("  %s rejected: %s\n", path, error.c_str());
    }
    return std::nullopt;
}

int main() {
    std::printf("crc32(\"123456789\") = %08X (expected CBF43926)\n",
                crc32(reinterpret_cast<const std::uint8_t*>("123456789"), 9));
    World world = {{1, 0, 0, 100, 55, kNoTarget}, {2, 5, 3, 80, 60, 1}, {3, -4, 8, 250, 0, 1}};
    save(world);                                       // first save
    world[0].health = 70;                              // the player takes a hit
    save(world);                                       // second save; the first becomes save.bak
    std::printf("clean load:\n"); load();

    // Damage the file on purpose (the game itself only ever writes through save()).
    Bytes damaged = readFile("save.dat");
    damaged[30] ^= 0x01;                               // one flipped bit on disk
    overwrite("save.dat", damaged);
    std::printf("one bit flipped:\n"); load();

    Bytes truncated = readFile("save.bak");
    truncated.resize(40);                              // a torn write from an in-place save
    overwrite("save.dat", truncated);
    std::printf("truncated:\n"); load();

    world[1].targetId = 9;                             // a reference to an object that was never saved
    Bytes oldFormat = encode(world, 1);                // a v1 save: no stamina field
    overwrite("save.dat", oldFormat);
    std::printf("v1 file with a dangling reference:\n"); load();
}

The output walks through four loads. The clean load finds health 70 and stamina 55. One flipped bit fails the checksum, so the loader falls back to save.bak, the previous save, with health 100. A file cut off at 40 bytes fails the length check and falls back the same way. A v1 file passes every check and loads with stamina defaulted to 100, and the guard's reference to object 9, which was never saved, comes back as no target. A crash between the two renames leaves no save.dat at all; the loader then falls back to save.bak, and the complete save.tmp is left behind for a smarter loader to try.

The widget is that file, byte for byte. Damage it one way, then press load:
flip a payload bit
flip a version bit
truncate to 40 bytes
write a v1 file
undamaged
load
save.dat: 92 bytesstored CRC EE12C89A, computed EE12C89Apress load
The program's save.dat, one cell per byte: the 16-byte header (magic, version, length, CRC-32) and then the payload. Pick a kind of damage, press load, and watch the checks run in the listing's order. A flipped payload bit leaves the header intact and is caught by the checksum; the readout shows the stored and recomputed CRCs. A flipped version bit turns version 2 into 6, which this build doesn't know. Truncating fails the length check. A v1 file passes every check and loads with stamina defaulted to 100. Whenever save.dat fails, the loader moves on to save.bak and loads the previous save, with health 100 instead of 70.
What's intentionally missing

    Error reporting to the player. The loader prints why a file failed. A game tells the player the newest save was damaged and that it loaded the previous one, and keeps the damaged file for a bug report.
    More than one backup. One save.bak covers one bad save. Rolling several, or keeping one per save slot, covers a migration bug that writes several bad saves in a row. This program also rotates save.dat into save.bak without checking it, so after a fallback load the next save replaces the good backup with the damaged file; a game validates save.dat before rotating it, or keeps the damaged file under another name.
    Per-section checksums and a schema. One CRC over the whole payload says the file is damaged but not where. Large saves checksum each section and tag each field, as §5's tagged formats do.
    Compression and encryption. Put both after encoding and before the checksum, so the CRC covers what's actually on disk.
    Platform save APIs. Consoles and some stores provide their own save storage with their own commit rules; the temp-and-rename dance is for plain files.

