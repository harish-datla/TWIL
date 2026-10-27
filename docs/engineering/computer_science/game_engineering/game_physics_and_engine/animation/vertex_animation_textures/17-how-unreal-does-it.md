# How Unreal Does It

Unreal’s AnimToTexture plugin supports vertex mode—positions and normals—and bone mode, where textures store bone transforms and weights are used in the material. Houdini plus SideFX Labs VAT 3.0 is the common path for cloth, shatters, splashes, fluids, and other baked simulations.

The City Sample uses a two-tier crowd: nearby pedestrians are fully rigged MetaHumans; distant pedestrians are vertex-animated static meshes from the same source animations. Niagara mesh particles can also play VAT animations with per-particle time offsets.
