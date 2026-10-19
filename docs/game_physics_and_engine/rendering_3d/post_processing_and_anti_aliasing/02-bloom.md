# Bloom

Bloom simulates light scattering around bright sources. A typical pipeline extracts bright pixels, progressively downsamples them, progressively upsamples with additive blending, then adds the result to the HDR image before tone mapping. A multi-scale pyramid gives a long glow at lower cost than a single wide blur.

One heavily downsampled blur is blocky when upscaled and pulses when a bright highlight crosses coarse texels. On the first downsample, use a Karis average to suppress fireflies: weight each 2×2 group by `1/(1+luma)` before normalizing. A bright subpixel then contributes less to the mip rather than flooding the whole halo.

```glsl
vec3 down = e*0.125
    + (a+c+g+i)*0.03125
    + (b+d+f+h)*0.0625
    + (j+k+l+m)*0.125;
// First level: weight each 2×2 group by 1/(1+luma), then normalize.
vec3 up = (e*4.0 + (b+d+f+h)*2.0 + (a+c+g+i)) / 16.0;
```

Five halvings of 1920×1080 produce 960×540 through 60×34, totaling about one third of the full-frame pixels. This is far cheaper than a 64-pixel-radius full-resolution Gaussian.
