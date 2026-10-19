# A Short History of Getting Fast at I/O


Why does a 2026 PC game need a GPU decompressor and a special Windows read path just to load its textures? Because every console generation reinvented streaming against tighter constraints, and today's design keeps a piece from each one. The timeline is optional context; nothing later on the page depends on it.
Timeline: from CD-ROM loading screens (1996) to DirectStorage 1.4 (2026)

The short version: the virtual-texture layout dates from 2008[1] and the GPU-friendly codec from 2022[10]. What changed is how much of the pipeline the platform provides. On PC in 2026 the OS (IoRing, BypassIO), the driver (decompression metacommands, hooks where a GPU vendor swaps in its own decoder) and the GPU (compute-shader decode) each have a path built for streaming, where a decade earlier the engine did all of it on the CPU.

[Back to File Streaming for Game Engines](index.md)
