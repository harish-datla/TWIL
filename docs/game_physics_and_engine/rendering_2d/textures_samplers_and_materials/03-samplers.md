# Samplers

`VkSampler` is separate from the image and can be reused across images. It stores minification and magnification filters, mipmap mode, addressing outside 0..1 (repeat, clamp, mirror), and optional anisotropy.

For texel values `0, 100, 200, 50`, sampling at coordinate 1.75 selects 100 with nearest filtering. Linear filtering blends the centers at 1.5 and 2.5: `100 × 0.75 + 200 × 0.25 = 125`. Bilinear performs that in both axes over four texels.

Trilinear filtering chooses and blends between mip levels but uses one isotropic footprint. A floor footprint one texel wide and eight texels long can therefore blur the short direction eight times too much. Anisotropic filtering chooses the mip from the short axis and takes multiple probes along the long axis.

Anisotropy is optional. Enable `samplerAnisotropy` at device creation and clamp `maxAnisotropy` to the device’s `maxSamplerAnisotropy`; hard-coding 16 is invalid on weaker hardware. “Trilinear” means `minFilter = LINEAR` and `mipmapMode = LINEAR`. Nearest mipmap mode produces visible level-switching bands.
