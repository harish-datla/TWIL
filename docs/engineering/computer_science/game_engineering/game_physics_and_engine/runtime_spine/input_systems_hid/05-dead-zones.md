# Dead Zones


Leave a worn controller on the desk and the character slowly walks off by itself. A thumbstick never rests at exactly zero, so you ignore a small region near the center, the dead zone. The shape of that region is a commonly botched piece of gamepad code, and Josh Sutphin's article is the usual reference[10].

    Axial (per-axis): zero each axis independently. This makes a square dead zone that clips diagonals: a gentle diagonal can fall inside the square on both axes and read as zero, or snap to a cardinal direction (straight up, down, left or right). Correct only for grid or 4-way movement.
    Radial: zero by the vector's length. A circular dead zone, no cardinal snapping, but the output jumps from 0 to a finite value the instant you cross the boundary.
    Scaled radial (the right default for free aim): after the radial cut, re-scale so output ramps from 0 at the boundary to 1 at full deflection.

Worked example

Use a dead zone of 0.25, the widget's default. XInput's recommended left-stick value, 7849 out of 32767, works out to 0.24, a starting point to tune rather than a rule[7]. Push the stick gently diagonal to (0.20, 0.20). Its length is 0.28, past the 0.25 radius, so the player clearly meant to move.

    Axial: each axis is 0.20, under 0.25, so both get zeroed. Output (0, 0): the push is ignored. At (0.30, 0.10) only y is zeroed, giving (0.30, 0): the diagonal snaps to straight right.
    Radial: 0.28 is outside the circle, so the raw (0.20, 0.20) passes through, length 0.28. Pull back to a length of 0.24 and the output is 0. It never takes a length between 0 and 0.25.
    Scaled radial: (0.28 − 0.25) / (1 − 0.25) = 0.04, pointing the same diagonal way: (0.03, 0.03). Movement starts from zero right at the edge.

The scaled-radial rule as a formula. It keeps the stick's direction and rebuilds its length so the output runs from 0 at the dead-zone edge to 1 at full deflection:
eq. 1 · scaled-radial dead zone vout = v̂ · ( |v| − deadzone1 − deadzone )

In words: take how far the stick is pushed, subtract the dead-zone radius, and divide by what's left of the range, so the length ramps from 0 right at the boundary to 1 at full deflection. Multiplying by the unit direction v̂ keeps the aim direction while the fraction supplies the new length. Without the divide the stick would jump straight to a finite value at the boundary; the re-scale is what lets you walk slowly. Clamp |v| to 1 first: many sticks report diagonals slightly outside the unit circle (the 1.03 in §4), which would otherwise produce outputs above 1. Hover any symbol to see what it stands for.
Two traps: the square clips diagonals, and you must re-scale

The per-axis square dead zone is the classic bug. Even with a radial cut, forgetting to re-scale means the stick jumps to a finite output the moment it leaves the dead zone, so slow movement is impossible. The formula above fixes both.

The widget runs all three rules on a live stick. Drag the blue dot to the example's (0.20, 0.20) and switch shapes:
Axial (square)
Radial
Scaled radial
Dead zone 0.25
The blue dot and line are the raw stick, the green line is the shaped output, and the red square or circle is the dead zone. The readout beside the stick (below it on a phone) gives both vectors and their lengths; let go and the stick orbits on its own. In Axial mode, raw (0.20, 0.20) reads output (0.00, 0.00), and (0.30, 0.10) snaps to (0.30, 0.00), straight right. Radial passes (0.20, 0.20) through untouched, but a raw length of 0.26 gives 0.26 and 0.24 gives 0.00: the output jumps at the boundary. Scaled radial turns the same (0.20, 0.20) into (0.03, 0.03), length 0.04, and ramps from 0 at the circle to 1 at the rim, which is what lets you walk slowly.
Check Your UnderstandingQ1 of 2

Your twin-stick game feels like it snaps to 8 directions and slow diagonal aim is impossible. Most likely cause?
A per-axis (square) dead zone, which clips diagonals; switch to a scaled-radial dead zone.
The sticks need a larger dead zone.
Response curve set too steep.
0/2 correct

Wrong answers, and why: a bigger square dead zone worsens diagonal clipping; a response curve changes sensitivity, not snapping; and both scancodes and keycodes carry up/down, they differ in physical-vs-symbolic meaning.

[Back to Input Systems (HID)](index.md)
