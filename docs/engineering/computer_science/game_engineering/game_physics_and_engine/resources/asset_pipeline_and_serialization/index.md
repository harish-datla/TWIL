# Asset Pipeline & Serialization

Your game decodes PNGs every time a level loads. Renaming one texture breaks every material that pointed at it, last month's files stop loading after you add a field, and a header that loads fine on PC crashes on a phone. All four are asset-pipeline problems. The asset pipeline is the offline machine that turns authoring files like a .png or an .fbx into blobs the engine can load fast, often in place. This tutorial builds that machine in C++ and Rust: the cook step (the offline conversion from source file to runtime data), a versioned binary format, stable references, a dependency graph, hot reload, and the alignment rules that make zero-copy loading, using the file's bytes in place with no parse, actually work.
Time
~55 min
Level
Mid
Prereqs
Helpful, not required: File Streaming (how bytes get off disk) and Bit Shifting (for the binary layout).
Stack
C++ & Rust
◂ Build a Game Engine
Phase 4 · Resources
Next · Data Compression ▸
What you'll be able to do by the end

    Explain why engines cook source assets offline, and what a cooked texture looks like next to its PNG.
    Lay out a binary header with fixed-width fields, explicit padding and a documented byte order, and read it without an alignment crash.
    Version a format and migrate old files on load, or choose schema evolution instead.
    Reference assets by GUID on disk and by generational handle at runtime, and re-cook only what an edit touches.
    Package assets with an index, hot-reload them safely, and know the conditions zero-copy loading needs.


## Sections

- [Why a Pipeline](01-why-a-pipeline.md)
- [Bake vs Runtime Load](02-bake-vs-runtime-load.md)
- [Binary Serialization](03-binary-serialization.md)
- [Versioning & Migration](04-versioning-and-migration.md)
- [References & Dependencies](05-references-and-dependencies.md)
- [Packaging](06-packaging.md)
- [Hot Reload](07-hot-reload.md)
- [Zero-Copy Loading](08-zero-copy-loading.md)
- [Pitfalls](09-pitfalls.md)
- [What's Next](10-whats-next.md)
