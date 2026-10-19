# Telling the model where each token is

"dog bites man" and "man bites dog" contain the same tokens. Attention as described so far can't tell them apart: shuffle the input rows of an unmasked attention layer and the output rows come back shuffled the same way, with the same values. That property is called being equivariant to reordering, and it means word order has to be put in some other way.

The original transformer adds a position vector to each token's embedding. Other designs put position inside the attention calculation itself.[2] The causal mask also imposes an order, since each position sees a different prefix, but that limits what each position can see; it isn't an explicit position signal.

Rotary position embedding (RoPE) is one way to do the second. It splits each query and key into pairs of coordinates and rotates each pair by an angle proportional to the token's position, θ radians per position for that pair. Rotation never changes a vector's length. What changes is the angle between a query and a key, and that angle depends only on how far apart the two tokens are.[3]

Worked example

Query = key = (1, 0), θ = 0.4 radians per position.

Positions 3 and 5: the query turns by 1.2 radians, the key by 2.0. They end up 0.8 radians apart, so the dot product is cos 0.8 = 0.697.

Positions 13 and 15: turns of 5.2 and 6.0, still 0.8 apart, still 0.697. Positions 3 and 7: 1.6 radians apart, and the dot product drops to −0.029.

The equation says the same thing for any query q at position m and key k at position n, with R(angle) a 2D rotation.
(R(mθ)q)ᵀ(R(nθ)k) = qᵀR((n−m)θ)k

In words: rotating the query by its position and the key by its position gives the same score as leaving the query alone and rotating the key by the gap between them. Only n − m survives, so the score depends on relative position. A full head uses many coordinate pairs, each with its own θ.
Separate absolute position from relative gap
First position
3
Position gap
2
Pair frequency
0.4
Reset
Positions 3 and 5; relative gap 2. Rotated dot -0.409979; relative-only dot -0.409979; difference 3.3e-16.

Change the first position, the gap and the pair's frequency θ. The left panel shows the rotated query and key. The right-hand line recomputes their dot product as the first position slides from 0 to 20 with the gap held fixed, so it stays flat; changing the gap or the frequency moves it up or down. At the defaults (positions 3 and 5, θ = 0.4) the score is −0.410 computed either way, and the two results differ by about 10⁻¹⁶. A single pair oscillates as the gap grows, so this demo doesn't show attention fading with distance or any limit on context length.
Why the identity holds

RoPE doesn't give a trained model unlimited context. The formula accepts any position, but a model can behave poorly at positions far beyond the lengths it was trained on. A cache also has to use the same position numbering as the original pass: restarting positions at zero for a cached query changes its relation to every stored key.

In a full model the vectors being rotated already depend on their context. The identity is about rotating fixed vectors; it doesn't claim that moving words around leaves those vectors unchanged.
