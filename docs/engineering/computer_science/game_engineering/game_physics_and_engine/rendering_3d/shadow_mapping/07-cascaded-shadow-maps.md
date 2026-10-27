# Cascaded Shadow Maps

One directional shadow map cannot provide centimeter detail near the camera and still cover a distant valley. Cascaded shadow maps split the camera frustum into depth ranges and render each slice into a tightly fitted orthographic map. Near cascades cover less area at higher texel density; far cascades cover more at lower density.

Splits commonly blend uniform and logarithmic spacing. Two artifacts need explicit fixes:

- **Seams:** resolutions differ at boundaries; blend samples across a band.
- **Crawling:** the camera moves the projection by fractions of a texel; snap orthographic bounds to texel increments and pad the fit so the projection size stays constant.

Bias scales with each cascade’s texel size. Three or four cascades are common. Select by view-space depth or by testing which cascade contains the fragment. A single 2048² map over 100 m gives about 4.9 cm texels everywhere; a near cascade over 10 m gives about 4.9 mm.
