# Bake vs Runtime Load


Iterating on a level means loading it hundreds of times, so any work the load repeats is paid hundreds of times. The cook (or bake) is an offline transform from source to a runtime-ready, often platform-specific artifact: import → process → cook → package, which Gregory calls the asset conditioning pipeline[1]. Heavy work happens once, offline; the runtime does a light load and fixup, a small pass that turns stored offsets into real pointers[11]. The Bitsquid engine compiled its JSON source data into runtime data in a separate folder, and once running, the engine "never sees the generic data folder so it can't 'cheat' and read anything there"[3].
Cooked is not the same as compressed

Cooking is transform plus layout; compression is a separate, optional packaging step (its own tutorial). A cooked blob can be uncompressed. And "cook once, run anywhere" fails for platform-specific cooks: texture block format, endianness, and alignment can differ per target, which is why Unreal and Unity both build per target platform[2][14]. Keep import (source → engine intermediate) distinct from cook (intermediate → shipped package).

Worked example

Follow a 1024 × 1024 hero.png through the stages. Import decodes it to 4 MiB of RGBA. Process adds the mip chain (5.3 MiB) and compresses it to BC7, 1.3 MiB with mips. Cook writes that as a runtime blob with a small header, and package appends it to assets.pak at some offset, recorded in the index. At runtime, load is one 1.3 MiB read from that offset. Everything before it ran once, on the build machine.

One asset flows through the pipeline below; hit "edit the source" to trigger a hot re-cook of just that asset:
Bake hero.png
Edit the source (hot reload)
Six boxes are the stages, and the label under the lit box says what the asset is at that point. Press Bake to run it, then Edit the source. The source PNG is decoded, gets a mip chain, is transcoded to a GPU block format, written into a package at an offset, then loaded as a resident blob. Each stage transforms the representation. Hot reload re-cooks only the edited asset and swaps the result into the live scene, the payoff of §7.

[Back to Asset Pipeline & Serialization](index.md)
