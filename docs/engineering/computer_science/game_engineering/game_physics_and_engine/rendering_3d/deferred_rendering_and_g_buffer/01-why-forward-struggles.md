# Why Forward Struggles

Naive forward shading does work proportional to shaded fragments × lights. Overdraw makes it worse: a fragment may be lit expensively before a nearer object overwrites it. A 1920×1080 frame has 2,073,600 pixels. At 2.5× overdraw and 100 lights, naive forward performs about 518 million light evaluations; lighting only the visible surface would perform 207 million, before local light culling reduces it further.

This criticism is specifically about naive forward. Real forward renderers use per-object light lists and depth pre-passes, and Forward+ adds tiled/clustered culling. Deferred is one way to make the visible-surface property explicit.
