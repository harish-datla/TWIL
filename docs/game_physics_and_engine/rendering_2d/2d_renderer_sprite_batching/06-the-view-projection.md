# The View-Projection

The camera moves every frame, so the shader needs a fresh view-projection matrix. A `mat4` is 64 bytes and fits comfortably in Vulkan’s guaranteed minimum `maxPushConstantsSize` of 128 bytes. Push the premultiplied view-projection rather than separate model, view, and projection matrices; three matrices would be 192 bytes and overflow the pre-1.4 minimum on some mobile drivers.

Query `maxPushConstantsSize`; do not assume a larger limit. Stage flags in `VkPushConstantRange` must include every shader stage that reads the value. The remaining 64 bytes of a 128-byte range can hold small values such as tint or time.
