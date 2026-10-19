# Pitfalls

## Texture warps along a seam

Linear screen-space UV interpolation is wrong under perspective. Interpolate `attribute / w` and `1 / w`, then divide per pixel.

## Hard-coding warp size to 32

Warp and wavefront sizes are vendor- and architecture-specific: NVIDIA commonly uses 32; AMD supports 32 or 64. Query or design for the target instead of assuming.

## “Depth needs perspective correction”

NDC depth is already divided by `w` and is screen-linear. Other attributes need the perspective-correct divide.

## `discard` is free

Discard and alpha testing can delay depth writes until after the shader. Writing depth disables early testing unless conservative-depth guarantees apply.

## Thin triangles are cheap

2×2 helper lanes waste 25–75% of edge shading, and small triangles also pay setup costs. Use appropriate LODs and avoid excessive screen-space fragmentation.

## Far-plane z-fighting

Perspective depth concentrates precision near the camera. Use a floating-point reversed-Z buffer and avoid coplanar surfaces where possible.

## Splitting passes on mobile

Sampling an attachment after ending a render pass can force a round trip through main memory on a tiler. Prefer subpasses or local reads when the data is only an intermediate.
