# Encoding Positions: The Bounding-Box Trick

Most VAT textures store values in [0,1]. Compute the minimum and maximum position over every vertex and frame, then encode:

```text
pixel = (position − bboxMin) / (bboxMax − bboxMin)
position = pixel × (bboxMax − bboxMin) + bboxMin
```

For a 2 m box, 8-bit storage has 2/255 ≈ 7.8 mm per step and up to 3.9 mm half-step error. Half floats provide roughly 1 mm worst-case in that normalized range. Split two 8-bit textures can provide near 16-bit fixed-point precision on devices without float textures. R32 is rarely needed; BC6H is compact but lossy and its 4×4 blocks span unrelated vertices and frames, so test every asset.

Use linear texture imports. Positions are data, not color. Recompute the box across all frames or peak positions will clamp to its boundary.
