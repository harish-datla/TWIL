# Early-Z

The logical pipeline places depth testing after the fragment shader, but opaque scenes often overdraw heavily. Hardware can run depth testing before the shader when it can prove the result is functionally identical.

At 1080p with three opaque layers per pixel, a late test runs the fragment shader 6.2 million times and discards two thirds. Drawing front-to-back with early-Z can shade only about 2.1 million visible fragments. A depth pre-pass can provide the same benefit when draw ordering is difficult.

Early-Z is an optimization, not a stage to rely on. If a shader writes depth, the value does not exist early, so the early test is disabled. `discard`, alpha testing, and alpha-to-coverage may prevent early depth writes because the shader can still kill a sample. Ordinary texture reads and branches do not inherently disable early-Z. Conservative depth output can preserve early rejection when the shader promises its depth only moves in one direction.
