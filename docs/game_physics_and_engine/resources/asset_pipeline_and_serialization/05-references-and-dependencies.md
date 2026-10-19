# References & Dependencies


An artist renames granite.png to granite_01.png, and every material that used it renders with the missing-texture placeholder. Assets point at other assets, and how you encode that reference decides whether moving a file breaks the game.

    GUID = stable identity. A global ID that survives move and rename. Unity stores one per asset in a sidecar .meta so it "can move or rename the asset without breaking anything"; lose the .meta and "any reference to that asset is broken"[9]. A path, by contrast, breaks the moment a file moves.
    Runtime handle ≠ GUID. At load, GUIDs resolve to compact runtime handles. A common shape is a generational handle (index + generation), so a freed slot reused by a new asset doesn't alias an old handle[10]. This is the same pattern as entity IDs in the ECS (entity component system) and the handles in Memory Allocators.
    Dependency graph. The cooker records which assets each asset references; in the well-behaved case that forms a DAG, a directed acyclic graph whose arrows never loop back. Touch a source asset and it, plus every asset that depends on it, must re-cook.

Worked example

Slot 7 of the mesh table holds rock.mesh at generation 3, so the handle is (7, 3). The level unloads the rock; slot 7 is freed and reused for tree.mesh, and its generation becomes 4. A system still holding (7, 3) looks up slot 7, sees generation 4, and gets "stale handle" instead of silently drawing a tree where the rock was. The GUID in the level file never changed; only the runtime handle did.

In the widget, arrows point from an asset to the assets that use it, the direction a change spreads. Click an asset (or focus the graph and use the arrow keys and Enter) to "edit" it; the dirty set, the edited asset plus everything that depends on it, propagates downstream:
Reset (all clean)
Click an asset to edit it; its dependents must re-cook.
Green assets are clean and orange ones must re-cook; the line above the graph counts them. Editing shader.inc reads "dirties 4 dependents": the shader, the material, the mesh, and the prefab, everything downstream, while rock.png stays clean. Editing rock.png dirties 3. That's incremental cooking: touch one file, re-cook only what depends on it. The traversal keeps a visited set, so a reference cycle, which real projects do end up with, can't loop forever.
Check Your UnderstandingQ1 of 2

What is the difference between an asset GUID and a runtime handle?
A GUID is a stable, persisted global identity; a handle is an ephemeral runtime index (ideally generational) that can be reused.
They are the same thing with different names.
A handle is global and persisted; a GUID is the runtime index.
0/2 correct

Wrong answers, and why: GUID and handle are opposite roles (persisted identity vs runtime index), not synonyms; and a cast crash that's platform-specific is an alignment/layout problem, not memory or mmap availability.

[Back to Asset Pipeline & Serialization](index.md)
