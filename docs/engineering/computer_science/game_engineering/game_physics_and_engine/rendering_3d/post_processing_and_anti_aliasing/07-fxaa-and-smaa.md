# FXAA and SMAA

Post-process AA works on the finished image, which is useful when a deferred renderer cannot afford a multisample G-buffer. FXAA converts local color to luminance, detects contrast edges, and blends along them in one LDR pass. SMAA adds morphological edge detection and pattern classification for straighter, sharper blends.

These methods are spatial and lossy. FXAA cannot distinguish a geometric edge from a high-contrast texture feature, so it can blur real detail. Neither basic method knows history, so temporal crawl remains. SMAA’s temporal variants add reprojection.
