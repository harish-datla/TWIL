# Comparing Floats


A unit test checks that a transformed point lands where it should, and fails with "expected 1048576, got 1048576.25". They're only two floats apart. a == b on floats is exact equality, and after any rounding or reordering that's usually not what you want. There are three tolerance strategies, and each fails somewhere[7].

    Absolute (|a−b| ≤ ε, with ε a fixed small number): works near zero, fails at large magnitudes, where a fixed ε is smaller than one ULP and the test degrades into ==.
    Relative (|a−b| ≤ ε·max(|a|,|b|), so the tolerance scales with the values): works across magnitudes, breaks down near zero, and is meaningless across zero (opposite signs).
    ULP-based: reinterpret the bits as integers and subtract; the result is the count of representable floats between them. Intuitive, but undefined across signs and gigantic when comparing a tiny number to 0.0.

Worked example

At 1,048,576 (220) one ULP is 2−3 = 0.125, so a = 1,048,576 and b = 1,048,576.25 are 2 ULP apart, |a − b| = 0.25.

    Absolute with ε = 10−6: 0.25 > 0.000001, so not equal. At this size even two neighboring floats can't pass.
    Relative with ε = 10−5: 0.25 / 1,048,576.25 ≈ 2.38 × 10−7 ≤ 10−5, so equal.
    ULP with a limit of 4: 2 ≤ 4, so equal.

Dawson's practical answer: combine an absolute "safety net" for the near-zero case with a relative or ULP comparison otherwise, and pick the absolute tolerance from the scale of your problem. His summary is worth memorizing: "zero is a huge nuisance, avoid it if possible"[7].

The widget runs all three tests on two floats a few ULP apart, at every magnitude from 2−24 to 224:
Magnitude 2^ 20
Gap 2 ULP
Each bar is one test swept across magnitude: green where it calls the two floats equal, red where it doesn't. The yellow marker and the readout below follow the magnitude slider; the gap slider sets how many ULP apart the floats are. The default is the worked example: at 220 the readout shows |a−b| = 2.50e-1 and relative = 2.38e-7, and only the absolute test says NOT equal. Slide the magnitude down and the absolute test (ε = 1e-6) passes at 22, where |a−b| = 9.54e-7, and fails from 23 up, where it's 1.91e-6: past that point ε is smaller than the gap, and the test degrades toward ==. Relative (1e-5) and ULP (up to 4 ULP) stay green because they scale with the values, though relative still breaks down comparing against zero and ULP is undefined across signs, the cases the list above warns about. Push the gap past 4 ULP and the ULP bar turns red at every magnitude: its threshold is a count of floats, not a distance.

[Back to IEEE-754 Floating Point](index.md)
