# Frame Interpolation

If a game runs at 60 fps and a bake is 30 fps, point sampling holds each baked frame for two display frames. Linear interpolation uses the fractional frame:

```text
P = (1 − α) × P_floor + α × P_ceil
```

Bilinear V filtering performs this when rows are correctly centered. Manual two-fetch interpolation is safer for padding, stacked clips, and explicit loop handling. Cubic/Catmull–Rom uses four samples and smooths velocity, but cannot recreate motion absent from the bake.

Loop with REPEAT only when the texture contains exactly the clip rows. Otherwise wrap frame indices manually or append frame zero after the final row. Never interpolate across animation cuts; clamp inside each clip’s range.
