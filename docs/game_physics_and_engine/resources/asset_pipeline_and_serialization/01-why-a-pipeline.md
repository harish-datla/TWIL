# Why a Pipeline


Every time a level loads, your engine decodes its PNGs on the CPU, builds their mip chains and converts them for the GPU, and does exactly the same work again on the next load. Source formats are built for authoring and interchange, not for runtime. A PNG must be decoded before a GPU can sample it; a WAV must be parsed; an FBX file, Autodesk's 3D interchange format, carries a whole editor scene that has to be converted. Even glTF, Khronos's open 3D format, whose binary buffers are designed to reach the GPU with little processing, stores textures as PNG or JPEG and uses its own vertex layout rather than the engine's. Unreal puts it plainly: it stores content "in particular formats which it uses internally, such as PNG for texture data or WAV for audio. However, this content needs to be converted to different formats for the various platforms"[2].

So an engine ships a second representation: cooked data, laid out for fast loading and the target hardware. The conversion also generates mip chains (each texture's progressively half-size copies), transcodes textures to GPU block formats (fixed-rate compressed formats the GPU samples directly), and lays the data out for the platform.

Worked example

Decoded, a 2048 × 2048 albedo PNG is 2048 × 2048 × 4 bytes = 16 MiB of RGBA pixels, plus another third for its mip chain, and the CPU has to produce all of it before the GPU sees a pixel. The cooked BC7 version is 1 byte per pixel: 4 MiB, or about 5.3 MiB with mips. The GPU samples it directly, so at load the engine reads 5.3 MiB and hands it over, with no decode and no mip generation.
There is no single right format

The design space splits along axes: text vs binary, parse vs zero-copy, a version tag vs schema evolution, GUIDs vs paths. The right choice depends on your platforms, your iteration speed, and whether data crosses a version boundary at runtime. This tutorial builds one coherent point in that space and names the tradeoffs at each axis.

[Back to Asset Pipeline & Serialization](index.md)
