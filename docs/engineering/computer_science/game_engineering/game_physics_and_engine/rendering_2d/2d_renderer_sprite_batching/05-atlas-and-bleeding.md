# Atlas and Bleeding

An atlas packs sprites into one image so they share a descriptor and do not break a batch. Texture arrays and bindless descriptors are alternatives when their constraints and features fit.

Bilinear filtering samples four neighboring texels. At an atlas boundary it can pull in the next sprite, creating a seam. Use a half-texel UV inset, a padding gutter (at least a couple of pixels), and edge extrusion. Mipmaps make bleed worse because every level halves the gutter. A gutter must grow with the coarsest mip, or use per-sprite mip chains or a texture array.

If a sprite occupies texels 0–7 in a 256-texel atlas, its UV edge is `8/256 = 0.03125`; sampling exactly there lies halfway between texel 7 and neighbor 8. Inset to half-texel centers (`0.5/256` and `7.5/256`) and extrude the edge. A two-texel mip-0 gutter becomes one texel at mip 1 and half a texel at mip 2.
