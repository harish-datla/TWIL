# Exposure

A tunnel and daylight can differ by dozens of times in luminance, so one fixed exposure cannot show both. Exposure multiplies HDR before tone mapping. Auto-exposure measures scene luminance, selects a target, and adapts toward it over time.

Use a log-average/geometric mean so a few bright pixels do not dominate. A histogram can additionally discard extreme percentiles. Smooth adaptation by `1 − exp(−dt × speed)`, usually in exposure values, or brief flashes will pump the whole image. Separate adaptation speeds for brightening and darkening often feel more natural.

If a scene moves from luminance 0.04 to 1.5, the 37.5× gap is 5.2 EV. A key value of 0.5 targets exposures 12.5 and 0.33. An arithmetic mean with a bright sign can pull exposure down by 100×; a log average changes much less.
