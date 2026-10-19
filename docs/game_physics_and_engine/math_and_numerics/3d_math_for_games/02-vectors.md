# Vectors


A guard needs to know whether the player is in front of it and inside its 90° view cone; a shader needs to know how squarely light hits a surface; a mesh tool needs a triangle's normal. A 3D vector is three numbers, but the two products you do with it carry most of that geometry: the dot product measures alignment, the cross product builds a perpendicular.

The dot product collapses two vectors to a single number that encodes the angle between them, which is what every "is it in front of me" test needs:
eq. 1 · dot product a · b = axbx + ayby + azbz = |a| |b| cos θ

In words: two routes to the same number. Left: multiply the matching components and add, pure arithmetic. Right: multiply the two lengths by the cosine of the angle between them. The right-hand form is why the dot product measures alignment: cos θ is 1 when the arrows point the same way, 0 when they are perpendicular, and negative once they point more than 90° apart. Hover any symbol to see what it stands for.

Its sign tells you which half-space b is in relative to a: positive in front, negative behind, zero perpendicular. That sign test powers CPU-side back-face checks, the N·L term in diffuse lighting (the surface normal dotted with the light direction), and "is the enemy in front of me" checks (a field-of-view test compares the dot product against the cosine of half the view angle).

Worked example

A guard faces f = (1, 0, 0) and has a 90° view cone, so half the angle is 45° and cos 45° = 0.707. The direction to the player, already normalized, is p = (0.6, 0.8, 0).

f · p = 1·0.6 + 0·0.8 + 0·0 = 0.6: positive, so the player is in front, but 0.6 < 0.707, so they're outside the cone (the angle is acos 0.6 ≈ 53°, more than 45°). Three multiplies, two adds and a compare; no trig at run time.

The cross product (3D only) returns a vector perpendicular to both inputs, with length equal to the area of the parallelogram they span. Cross two edges of a triangle and you get its normal:
eq. 2 · cross product a × b = (aybz − azby, azbx − axbz, axby − aybx)
|a × b| = |a| |b| sin θ

In words: the result is a new vector pointing perpendicular to both a and b, straight out of the plane they share. Each output component criss-crosses the other two axes; that pattern is the whole formula. Its length equals the area of the parallelogram a and b span, and because that area carries sin θ, the cross product is largest when the arrows are perpendicular and shrinks to zero when they are parallel.

Worked example

a = (2, 0, 0) and b = (1, 1, 0). The formula gives (0·0 − 0·1, 0·1 − 2·0, 2·1 − 0·1) = (0, 0, 2): straight along z, perpendicular to both.

Its length, 2, is the parallelogram's area: base |a| = 2 times height 1 (b rises 1 unit above a's line).
Trap: the cross product's direction depends on handedness, its formula does not

The component formula above is fixed. Whether the result points "up" or "down" relative to your screen depends on whether your basis is left- or right-handed. The right-hand rule only describes a right-handed basis; Unreal's coordinate system follows the left-hand rule and its docs say so explicitly[8]. Don't write "the cross product gives the right-hand-rule normal" without scoping it to the basis you're in.

Projection of a onto b drops a perpendicular from the tip of a onto the line through b. It answers "how much of this velocity points along that wall or that axis", a question character controllers and physics ask constantly:
eq. 3 · projection of a onto b projb(a) = a · bb · b b → (a · b) b when |b| = 1

In words: how much of a points along b? Drop a straight line from the tip of a onto b's direction; the shadow it casts is the projection. The fraction (a·b)/(b·b) works out to how many copies of b fit into that shadow, and multiplying by b aims the answer back along b. When b is already unit length the bottom is 1 and the whole thing collapses to (a·b) b.

Normalization scales a vector to unit length while keeping its direction. The view-cone test above only works on unit vectors, so this runs before most of them:
eq. 4 · normalize â = a|a|

In words: divide a vector by its own length and the direction survives but the length becomes exactly 1, a unit vector, written with a hat: â. Directions you compare or feed into lighting should almost always be normalized first. The trap: a zero-length vector has |a| = 0, so this divides by zero and produces a NaN that then spreads through every transform it touches, so guard against it.

Worked example

a = (3, 4, 0) has length √(9 + 16) = 5, so â = (0.6, 0.8, 0).

Project a onto b = (2, 0, 0): a·b = 6 and b·b = 4, so the shadow is 6/4 × (2, 0, 0) = (3, 0, 0). With the unit b̂ = (1, 0, 0) the short form gives the same: (a·b̂) b̂ = 3 × (1, 0, 0).

And when you actually need the angle between two vectors, acos(a·b) (for unit vectors) is numerically poor near 0 and π; atan2(|a×b|, a·b), using the two-argument arctangent, is the robust form.

Drag the blue arrow, or use the sliders. The dot product flips sign as b swings past 90°, and the parallelogram (the cross product's magnitude) swells to its largest when the arrows are perpendicular:
Angle of b 55°
Length of b 1.00
projection of b onto a
parallelogram (area = |a×b|)
Red a is fixed and blue b follows the sliders or your drag; the panel beside them reads out the angle, a·b, the projection and the cross product. At the starting 55° it shows a·b = 0.69 and an area of 0.98; drag b to 90° and a·b drops to 0.00 while the area peaks at 1.20 (a is 1.2 long). The dot product stays positive while b leans the same way as a, hits zero at 90°, and goes negative past it, exactly tracking the projection of b onto a (the dashed shadow flips to the far side). The shaded parallelogram is the cross product's magnitude: zero when the arrows line up, largest when perpendicular. The ⊙ / ⊗ glyph marks whether a×b points out of the screen or into it, the sign that handedness pins down.
A 3D vector, by hand
C++
Rust

struct Vec3 { float x, y, z; };

inline float dot(Vec3 a, Vec3 b) { return a.x*b.x + a.y*b.y + a.z*b.z; }

inline Vec3 cross(Vec3 a, Vec3 b) {
    return { a.y*b.z - a.z*b.y,      // x
             a.z*b.x - a.x*b.z,      // y
             a.x*b.y - a.y*b.x };    // z
}

inline float length(Vec3 v) { return std::sqrt(dot(v, v)); }

inline Vec3 normalize(Vec3 v) {
    float len = length(v);
    return len > 1e-8f ? Vec3{ v.x/len, v.y/len, v.z/len } : v;  // guard the zero vector
}
// In practice: glm::dot, glm::cross, glm::length, glm::normalize.


[Back to 3D Math for Games](index.md)
