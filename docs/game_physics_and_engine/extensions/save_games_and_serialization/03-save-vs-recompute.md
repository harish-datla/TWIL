# Save vs recompute

A save that includes the navmesh and every cached path is big, and it can disagree with itself after a patch. Persist authoritative state, the facts nothing else can reproduce; recompute derived state, anything you can calculate from those facts, on load. Saving derived state bloats the file and creates a second source of truth that can drift out of sync with the authoritative fields it was computed from.

    Persist: positions, health, inventory, quest flags, the RNG seed and stream position, and the world-mutation deltas from §4.
    Recompute: the navmesh and spatial acceleration structures, render and physics proxies, cached paths, animation runtime state, and derived UI. They reload with the level or rebuild from the authoritative data.

Re-spawn runtime objects, don't blind-serialize them

An object spawned during play (a dropped item, a summoned creature) is saved as the fact of its existence plus its authoritative fields, then re-spawned on load, not serialized as a live engine object with all its transient machinery. SPUD, an open-source Unreal persistence library, does exactly this: runtime-spawned actors "will be automatically re-spawned on load"[6]. Unreal guidance also warns against keeping persistent data in framework actors that are rebuilt with each level (GameMode, GameState, GameSession)[1]. The rule of thumb: if you can recompute it from what you saved, don't save it.

Worked example

A pack holds 12 items, and the save also stores their cached total weight, 42.5 kg. Patch 1.1 changes a sword from 3 kg to 3.5 kg. A player carrying two swords loads a save that still says 42.5 kg while the items now add up to 43.5 kg, and the encumbrance check uses whichever number it reads. Recompute the total from the item list on load and there's nothing to drift.
