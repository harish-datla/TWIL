# Full vs delta saves

A level has thousands of placed objects, and by the time the player saves they've changed a few dozen of them. The cooked level is already on disk, read-only and shared by every player. Re-storing all of it in each save is wasteful, so most level-based games persist only a delta over that baseline: the objects whose state changed from their authored default. Unreal's serializer has this built in, in its default mode "a property's value is not altered from its default... the property will not be serialized"[7].

Bethesda's Creation Engine is the canonical shipped example. A Skyrim save stores change forms, the deltas over the baked plugin data, and writes references as packed three-byte RefIDs whose type bits say whether the form comes from a plugin or was created at runtime (plugin index 0xFF) and exists only in that save[8]. The save also records the plugin list it was written against, so saved references can be mapped onto the current load order. Removing a mod mid-playthrough is what breaks saves: change forms and script state still reference records that no longer exist, a real-world dangling reference.

Worked example

Say a level has 5,000 placed objects with 48 bytes of state each: 240,000 bytes for a full snapshot. The player has opened 40 doors and looted 25 chests, so 65 objects differ from the cooked level. With an 8-byte id plus 48 bytes each, the delta is 65 × 56 = 3,640 bytes: about 1.5% of the snapshot. It grows only with what the player changes.
Delta is the common approach, not the only one

The practical delta is "changed from the class default," cheap to detect by comparing each object to its baseline. It is not a graph diff between two whole saves, which is expensive and rarely worth it; a single-scene or roguelike-run game legitimately snapshots its entire state, and an MMO keeps the authoritative state server-side. Match the strategy to the game, but know that for an open world the delta-over-baseline is what keeps each save small.

A delta only works if every object in the cooked level has an id that stays the same from build to build, because the save says "object 114: smashed", not "the barrel at row 2, column 3"; the Skyrim RefIDs above play that role. The id also exposes the removed-mod failure described above: when a patch or a removed mod deletes an object, the save still holds a change for it. The loader has to skip that record (and usually log it) instead of failing the whole load.
patch: remove object 114
reset
full snapshot 2,304 Bdelta 280 B (5 changes, 12%)on load: 5 applied
Each cell is one placed object from the cooked level (D, C or B for door, chest or barrel, with its id); the change list next to the grid is what the save stores. Click objects to open, loot or smash them, then press patch. At 48 bytes of state per object, the full snapshot is always 2,304 bytes; the five starting changes take 5 × 56 = 280 bytes, 12% of it, and the delta grows only as you click. The patch deletes object 114 from the level while the save still holds a change for it: the readout shows one record skipped on load and the rest applied.
