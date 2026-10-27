# How Unreal Does It


If you ship on Unreal, you're already using most of this page through its built-in systems, and knowing which one owns which asset tells you where to look when something pops in. Four of Unreal Engine 5's streaming systems map onto the sections above, and most projects use several at once.

    Texture Streaming.[34] The classic mip-based streamer: a texture's smallest mips load with it and stay resident, and its larger mips stream on demand. The streamer works out which mip each texture needs from its on-screen size, drops mips when the total is over budget, and submits async loads to bring up the rest. This covers ordinary, non-virtual textures.
    Streaming Virtual Texturing (SVT).[35] The SVT pattern from §10 wired into the material editor. A material can sample a virtual texture instead of a regular one; the engine builds the indirection table, runs the feedback pass, and pages fixed-size tiles in and out of a physical cache. Aimed at very large textures, UDIM sets (one surface's texture split across a grid of image tiles) and virtual-texture lightmaps. Its sibling, Runtime Virtual Texturing, renders its tiles on the GPU at runtime (typically for landscape materials) instead of streaming them from disk.
    Nanite.[9] The cluster-streaming geometry pipeline from §11. 128-triangle clusters, 128 KB pages, GPU cluster culling, software rasterizer for micropolygons, triangles about a pixel in size. The mesh equivalent of SVT.
    World Partition.[36] Replaces the legacy sub-level system. The world is a 2D grid of "cells"; actors are stored one-per-file under the grid; cells load within each runtime grid's loading range of a streaming source (by default, the player). HLODs (Hierarchical Levels of Detail) supply a coarse representation of far cells so the world looks complete even when only the local cells are resident.

The four layers in one frame

A typical Unreal frame might: page in actors via World Partition (large-grain spatial residency), sample materials through SVT (texel-level residency), render via Nanite (cluster-level residency), and use mip streaming for the textures that aren't virtual. Each layer runs independently at its own granularity: actors, texel tiles, cluster pages, whole mips.

[Back to File Streaming for Game Engines](index.md)
