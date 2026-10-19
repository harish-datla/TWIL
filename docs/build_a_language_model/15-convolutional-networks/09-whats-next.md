# What's next

Diffusion Models is the series' other adjacent page: the same training machinery used to generate images instead of classifying them. On the main path, Transformers and Attention replaces the fixed 3×3 neighborhood with mixing that's computed from the input. Each position decides how much to draw from every other position, using weights that are recomputed for every input, while the matrices that produce those weights are trained like any other parameters.
