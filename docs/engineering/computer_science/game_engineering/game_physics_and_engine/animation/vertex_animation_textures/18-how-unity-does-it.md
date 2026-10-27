# How Unity Does It

Unity has no single official AnimToTexture equivalent. Common options are Houdini/SideFX Labs VAT shaders, OpenVAT’s Blender baker and Unity package, and Shader Graph implementations that take UV1 and time and output displaced positions/normals. HDRP examples and Visual Effect Graph integrations cover mesh and sprite modes.

DOTS/Entities Graphics supplies per-entity material overrides, a natural place for each VAT instance’s time offset. Tooling varies, but runtime decode remains the same layout and bounding-box math.
