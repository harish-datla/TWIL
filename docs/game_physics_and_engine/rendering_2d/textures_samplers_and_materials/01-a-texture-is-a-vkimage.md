# A Texture Is a VkImage

A bilinear read fetches a 2×2 block of neighboring texels, so texture hardware wants texels close on screen to be close in memory. A `VkImage`, unlike a flat buffer, has a format, tiling, and layout. Optimal tiling is implementation-private—often tiled or Morton/Z-order—so nearby 2D texels can be fetched together.

For a 1024-wide RGBA8 image, a row-major 4×4 block spans four separate 16-byte pieces about 4 KiB apart. Morton order interleaves x and y bits, making each aligned 4×4 block a contiguous 64-byte run.

Do not confuse tiling with layout. Tiling (`OPTIMAL` versus `LINEAR`) describes memory arrangement; `VkImageLayout` describes the access state used for synchronization. Creating an image also does not allocate its memory. `vkCreateImage` provides requirements; allocation and binding are separate, though VMA combines them. Avoid one allocation per texture: the specification only guarantees a finite `maxMemoryAllocationCount` (at least 4096), so sub-allocate from larger blocks.
