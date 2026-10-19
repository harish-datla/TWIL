# MSAA

Multisample anti-aliasing tests coverage and depth at multiple sample positions but generally runs the pixel shader once per pixel/covered primitive. Resolve averages covered samples. It smooths geometric edges without quadrupling shading everywhere.

MSAA does not fix shader/specular aliasing. Alpha-tested cutouts need alpha-to-coverage. Deferred rendering is expensive with MSAA because the G-buffer needs per-sample storage and lighting at edge samples. At 4×, a 16-byte G-buffer becomes 64 bytes per pixel. This is why many deferred engines prefer post-AA or temporal AA.

MSAA is not SSAA: SSAA shades every sample and can reduce shading aliasing, but pays the full cost. At a 4× edge with three covered samples, resolve produces a 75% foreground and 25% background blend from one shader result.
