# The Conventions Are the Hard Part


A formula copied from a forum post renders your scene upside down; a model exported from another tool comes in mirrored. Neither is an algebra mistake. The vector and matrix algebra in this tutorial is the same algebra you saw in school. What makes 3D math for games its own skill is that a single rotation can be written several equally correct ways, and a formula copied from a source with different conventions renders upside down, mirrored, or inside out. The algebra doesn't change between engines and APIs; the conventions do, and most of the debugging time goes into them[1][2].

Four independent choices trip people up, and they are genuinely independent, so you can mix them in any combination:

    Storage order: row-major (rows stored one after another) or column-major (columns one after another). How the numbers sit in memory.
    Vector convention: column vectors (v′ = M·v, matrix on the left) or row vectors (v′ = v·M, matrix on the right).
    Handedness: left- or right-handed world space. Which way the cross product points.
    Clip space: where +Y ends up on screen and whether the depth range is [−1, 1] or [0, 1].

This tutorial picks one of each and holds it throughout: column-major storage, column vectors (v′ = M·v), right-handed world space, and Vulkan clip space. The first three are the defaults of the two libraries the code uses, glm for C++[3] and glam for Rust[4]. Vulkan clip space takes a little configuration in each (§5): glm defaults to OpenGL's depth range, and neither library flips Y for you. Stated once; every sample below obeys it.

Five widgets let you break each convention on purpose and watch what happens: the vector products, transform order, clip space and handedness, gimbal lock, and rotation blending.

[Back to 3D Math for Games](index.md)
