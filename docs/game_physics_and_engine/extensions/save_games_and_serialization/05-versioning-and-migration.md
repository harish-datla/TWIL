# Versioning & migration

Patch 1.1 adds a stamina stat, and the first player to load an old save gets a crash, or worse, a stamina value read from whatever bytes happen to follow the health field. Versioning is what breaks in production. You will ship, then patch the save format, and players still have saves written by the old code. They must keep loading. The mechanism is a version tag plus migration: store a version number in the save, and keep code that upgrades each older version to the next, the same pattern as database schema migrations. Unreal's package format works this way, with an engine version number plus per-feature custom versions[9].

Pick a save version and load it into the current (v3) game. Older saves migrate up: a missing field gets a default, a renamed field is mapped. New code reads every old save:
On-disk save version:
v1
v2
v3 (current)
Pick v1, v2 and v3 in turn and compare the on-disk fields on the left with the loaded record on the right. Load a v1 save into v3 code: hp maps to the renamed health, and stamina and perks (added in v2 and v3) are filled with defaults. That is backward compatibility (new code reading old saves), the must-have for any shipped game. A v3 save loads as-is.
A versioned read: default fields that did not exist in older saves
C++
Rust

constexpr uint32_t SAVE_VERSION = 3;       // bump on every format change

std::optional<PlayerState> loadPlayer(Reader& in) {
    uint32_t version = in.u32();           // the version tag is the FIRST thing in the blob
    if (version > SAVE_VERSION) return std::nullopt;  // written by a NEWER build: refuse, don't misread
    PlayerState player;
    player.health  = in.f32();             // every version (stored as "hp" in v1/v2; same bytes)
    player.stamina = version >= 2 ? in.f32() : 100.0f;       // added in v2 -> default for older saves
    player.perks   = version >= 3 ? in.readPerks() : Perks{}; // added in v3 -> default for older saves
    return player;                         // new code reads ANY old save by defaulting missing fields
}

Worked example

Using the reader above and the widget's values: a v1 save starts with the version, 1, then hp = 80. The v3 reader takes 80 as health from the same bytes, skips the stamina read because 1 < 2 and sets 100, and skips perks because 1 < 3 and sets an empty list. A v2 save carries stamina = 55 after hp, so the reader keeps 55 and defaults only the perks. A save written by a future v4 build fails the 4 > 3 check and is refused rather than misread. Now change the stamina default from 100 to 120 in patch 1.2: every v1 save quietly loads with 120 stamina, a value its player never had.
Tagged formats tolerate drift; the long-tail traps are what bite

Positional binary needs an explicit version branch like the code above. A tagged format (Protocol Buffers, FlatBuffers) buys both compatibility directions cheaply: old code ignores unknown fields, new code defaults missing ones. But the rules are strict, and breaking them is where saves actually die: protobuf says never reuse a field number and reserve deleted ones so they cannot be recycled into a type-mismatched field[10]; FlatBuffers adds new fields only at the end and never deletes (only deprecates)[11]. The subtlest trap: editing a default value after ship silently rewrites the meaning of every old save that relied on the old default. Unreal's GUID-based custom versions exist precisely so two engine branches can evolve the format on independent tracks without colliding on one integer[9].
