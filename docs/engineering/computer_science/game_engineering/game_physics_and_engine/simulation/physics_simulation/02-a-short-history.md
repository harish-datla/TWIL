# A short history

Why does almost every game physics engine solve contacts one at a time in a loop, when the textbook answer is to solve them all at once? Because the methods engines ship were chosen over two centuries of mechanics and two decades of game development for speed and stability, with exactness a lower priority.
Timeline: from Euler (1765) to Jolt Physics (2022)

The pieces arrived in this order: Euler's rigid-body equations (1765)[16], the GJK distance algorithm (1988)[2], Baraff's exact contact-force solver (1994)[3], and Catto's iterative solver with warm starting (2005)[5], which Box2D, Bullet and Jolt build on.
