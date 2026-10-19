# Color Grading

Color grading applies an artistic transform without repainting textures. A common implementation is a 3D LUT, often 16³ or 32³, indexed by input color and filtered across the eight neighboring entries. A 32³ RGBA8 LUT is 128 KiB.

Decide deliberately whether grading is scene-referred in linear HDR before tone mapping or display-referred in LDR after tone mapping. Pre-tone-map grading can preserve behavior across HDR and LDR output; post-tone-map grading is a classic film/LDR pipeline. Tone mapping has already compressed highlights, so post grading cannot recover information lost there.
