# Text Rendering

Text is the batcher again: a bitmap font is a glyph atlas plus per-glyph metrics. Draw one quad per glyph using its atlas rectangle. Advance moves the pen after a glyph; kerning adjusts specific pairs. A HUD string such as `SCORE 1200` uses one quad per visible glyph and can share the UI atlas draw.

Bitmap glyphs blur when enlarged beyond their baked size. Signed distance fields (SDF) store distance rather than coverage so a small atlas remains crisp under magnification. Plain SDFs can round sharp corners; multi-channel SDF (MSDF) preserves corners with three distance channels. Full shaping—bidirectional text, ligatures, and complex scripts—is a separate problem handled by a shaping library.
