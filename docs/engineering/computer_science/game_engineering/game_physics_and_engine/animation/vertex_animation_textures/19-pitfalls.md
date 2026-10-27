# Pitfalls

- **sRGB position texture:** VAT data is linear; sRGB conversion squashes decoded values toward the bounding-box minimum.
- **Generated mipmaps:** Mips average unrelated vertices and frames. Disable mip generation and streaming.
- **Bilinear filtering on X:** Center U on texels, or point-sample and manually blend frames.
- **Rest-pose tangent frames:** Use a rotation texture or object-space normal map for deforming surfaces.
- **Wrong bounding box:** Compute bounds over every frame or peaks clamp to a plane.
- **Loop seam:** Explicitly wrap frames, use exact REPEAT layout, or append frame zero.
- **Lerping across cuts:** Clamp interpolation within each clip.
- **One crowd time:** Add a per-instance random time offset.
- **Wrong normal transformation:** Rotate object-space normals with the instance’s inverse-transpose when needed.
