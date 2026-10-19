# The data flow

A teammate wants the physics step to write its debug lines straight into a GPU buffer. It would work today and make the sim impossible to run without a GPU tomorrow. How the subsystems connect: input feeds the sim; the sim produces a render description and audio commands; the renderer and audio thread consume them. Click a node (or focus the diagram and use the arrow keys) to read its role, and try the forbidden edge:
add a sim → GPU edge (forbidden)

Click a node, or focus the diagram and press the arrow keys, to read its role.
Click a node to read its role, then tick the forbidden edge. The clean shape: input → sim → (render description → renderer → GPU) and (audio commands → audio thread, lock-free). Enable the sim → GPU edge and it flashes red: that coupling costs headless tests, clean interpolation, and the shape netcode needs.
