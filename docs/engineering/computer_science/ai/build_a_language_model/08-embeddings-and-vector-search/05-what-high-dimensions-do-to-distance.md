# What high dimensions do to distance

Embeddings typically have hundreds of coordinates, and in that many dimensions distances behave in a way 2D intuition doesn't predict. For random points, a query's nearest and farthest neighbours end up at almost the same distance. When everything is about equally far away, "nearest" stops meaning much, and a better index can't change that, because the problem is in the data.

Worked example

Scatter random points evenly through a cube with sides of length 1. In d dimensions, the distance between two of them averages about √(d/6), while its standard deviation stays near 0.24 whatever d is.

In 16 dimensions that's about 1.6 ± 0.24: the spread is 15% of the typical distance. In 256 dimensions it's about 6.5 ± 0.24, under 4%. Nearly every point sits in a thin shell around the query.

Real embeddings are less extreme than random points. Their intrinsic dimension, the number of directions in which the data actually varies, is usually far lower than their coordinate count, and that's one reason graph indexes still work on them. The effect still sets the limits.
Nearest and farthest converge as dimension grows
Dimension 2ᵏ, k
4
Reset
500 uniform random points in 16 dimensions, one random query: nearest distance 1.002, farthest 2.336, ratio 2.33, mean 1.700. As the ratio approaches 1 every point is about equally far away and "nearest" carries less information.

Slide the dimension up and read the ratio of the farthest distance to the nearest. Five hundred points uniform in the unit cube, one query, one seed. In 2 dimensions the farthest point is about 80 times as far as the nearest; in 16 dimensions about 2.3 times; in 256 dimensions about 1.2. The mean distance grows like the square root of the dimension while the spread around it doesn't, so the contrast that a nearest-neighbour search depends on shrinks.
