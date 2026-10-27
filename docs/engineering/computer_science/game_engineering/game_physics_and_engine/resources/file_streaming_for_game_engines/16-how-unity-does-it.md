# How Unity Does It


On Unity you'll write more of the streamer yourself: it ships fewer built-in streaming systems than Unreal and leaves more of the assembly to the project.

    Mipmap Streaming.[37] The counterpart of Unreal's texture streaming: only the mip levels the camera actually needs are kept resident. The streamer picks the mip from the camera's position and each mesh's UV distribution metric (how densely its UVs cover its surface); explicit overrides are available through Texture2D.requestedMipmapLevel for procedurally-loaded content.
    AssetBundles and Addressables. The bundle format from §7 wrapped in two APIs. AssetBundles is the low-level interface; Addressables is the higher-level system that handles dependency resolution, async loading, and reference counting.
    Subscenes (Entities / DOTS). The Entities package, part of DOTS (Unity's data-oriented technology stack), supports "subscenes": prebaked archetype chunks that load as a unit. The cell-based answer for ECS-heavy projects, where game objects are IDs whose data sits in packed component arrays (an entity component system).

Unity doesn't ship a virtual-geometry equivalent of Nanite. Streaming Virtual Texturing exists as an experimental feature, usable from Shader Graph shaders in HDRP (Unity's High Definition Render Pipeline) but not from HDRP's built-in shaders. The practical toolkit is Mipmap Streaming for textures, Addressables for bundles, subscenes for partitioning large worlds, and priority logic the project writes itself.

[Back to File Streaming for Game Engines](index.md)
