# Coordinate Spaces & Handedness


A rock modeled in one tool shows up in your engine mirrored, with its faces culled. To see why, follow where a vertex goes. It passes through a chain of spaces: object space → world space → view (camera) space → clip space → NDC (normalized device coordinates, the cube the GPU clips against) → screen. Each arrow is one matrix, except the last two, which are the perspective divide and the viewport transform the GPU does for you.

Handedness is fixed by the basis orientation. With +X right and +Y up: if +Z points toward you, the system is right-handed; if +Z points into the screen, it's left-handed. Formally, it is the sign of the triple product (X × Y) · Z with the three axes written in a reference right-handed frame.

Worked example

Write Unity's axes in a right-handed reference frame where +z points toward the viewer. Its +X is right, (1, 0, 0); +Y is up, (0, 1, 0); its +Z is forward, into the screen, so (0, 0, −1).

X × Y = (0, 0, 1), and (0, 0, 1) · (0, 0, −1) = −1: left-handed. glTF's axes give +1. That one sign is the difference a glTF importer for Unity has to convert by flipping an axis.
Trap: "right-handed" alone doesn't pin down a coordinate system

You also have to say which axis is up and which is forward, and the units. Three shipping conventions, all different:

    glTF: right-handed, +Y up, +Z forward, meters, radians[7].
    Unity: left-handed, +Y up, +Z forward[9]; one unit is conventionally a meter.
    Unreal: left-handed, +Z up, +X forward, distances in centimeters[8].

Import a model across a handedness boundary without converting and one axis flips, which mirrors the model and inverts triangle winding, and therefore which faces get culled. Unreal's coordinate-system page tells anyone importing from Maya, Houdini, ZBrush, Blender, or 3ds Max to account for exactly this[8].

[Back to 3D Math for Games](index.md)
