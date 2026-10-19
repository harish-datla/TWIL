# The Swapchain

The GPU cannot draw directly into the image the display is scanning. A swapchain is a rotating set of presentable images tied to the surface. Choose a surface format, present mode, image count, and extent, then create an image view for each image.

Prefer `B8G8R8A8_SRGB` with the sRGB color space, but enumerate formats and fall back because the pair is not guaranteed. FIFO is the only guaranteed present mode; MAILBOX and IMMEDIATE must be queried. If `currentExtent` is concrete, use it. If it is `0xFFFFFFFF`, choose the framebuffer pixel size and clamp it to the surface limits. Request `minImageCount + 1`, clamping to a nonzero maximum.

Logical window size and framebuffer pixel size differ under display scaling. A 1280×720 window at 150% scaling can require a 1920×1080 swapchain. Three such 4-byte images use about 24.9 MB. Swapchain image count is also independent of frames-in-flight count.
