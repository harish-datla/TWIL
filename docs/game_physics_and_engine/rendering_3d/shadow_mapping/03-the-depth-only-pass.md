# The Depth-Only Pass

The shadow pass needs only the nearest depth, not shading. Under dynamic rendering, use a depth attachment with no color attachments and draw geometry using the light view-projection. The resulting depth image is sampled during the camera pass.

Rendering back faces can move self-shadowing acne onto unseen faces, but it is not universal: thin, single-sided, or non-watertight geometry may disappear from the map. Another valid approach is no culling plus slope-scaled hardware bias. Choose D16 when tight bounds provide enough precision and bandwidth matters; D32 provides more precision. A 2048² D32 map is about 16 MiB; D16 is about 8 MiB.
