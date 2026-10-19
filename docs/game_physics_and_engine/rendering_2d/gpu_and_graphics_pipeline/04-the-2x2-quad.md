# The 2×2 Quad

Hardware does not shade pixels one at a time. It shades 2×2 quads so it can estimate texture-coordinate derivatives (`ddx` and `ddy`) for mip selection. The derivatives come from subtracting neighboring lanes.

A 1024-texel texture whose `u` increases by `1/256` per screen pixel has `ddx(u)=1/256`; the footprint is four texels per pixel, so the sampler roughly selects mip `log₂(4)=2`.

All four lanes must run even when one or more samples lie outside the triangle. Those are helper pixels; their results are discarded after providing derivatives. A one-pixel triangle still shades four lanes, wasting 75%. A one-pixel-wide, ten-pixel-tall sliver touches five quads and shades 20 lanes for 10 visible pixels. Thin geometry is therefore expensive, which is one reason engines use LODs as meshes shrink on screen.

Edge quads can waste roughly 25–75% of their shading work. Overdraw, divergence, and fixed per-triangle setup add further cost.
