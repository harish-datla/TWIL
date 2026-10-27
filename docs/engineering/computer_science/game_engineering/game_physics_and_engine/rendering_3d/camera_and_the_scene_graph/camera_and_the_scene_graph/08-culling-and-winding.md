# Culling and Winding

Back-face culling skips triangles facing away from the camera. Set `cullMode = BACK` and choose the correct front-face winding. glTF declares counter-clockwise fronts.

Vulkan determines winding in framebuffer coordinates. A vertical mirror reverses it. If an OpenGL-style projection is used without the Vulkan Y flip, glTF’s CCW triangles appear clockwise on screen, so `BACK` culling discards the visible faces. Flip Y in the projection or use a negative viewport height, then `frontFace = COUNTER_CLOCKWISE` matches glTF again.

An extra flip can reverse it a second time, causing an inside-out mesh. The same idea is visible in a triangle’s signed area: mirroring Y changes a positive area to a negative one. An offscreen renderer that intentionally omits the flip may deliberately choose the opposite front-face setting.
