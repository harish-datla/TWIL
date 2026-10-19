# Summing a Million Numbers


A profiler adds up a million frame timings, or a stats screen totals the distance a player walked over a long session. Add a long list of floats naively and the running total grows large while each new term stays small, so each addition rounds away part of the term, and the error accumulates.

Kahan summation fixes this by keeping a second float, the compensation, that holds exactly what the last addition rounded away, and feeding it back into the next term[8].

Worked example

Start a float total at 16,777,216 (224), where floats are 2 apart, and add 1.0 twice. Naively, 16,777,216 + 1 is a tie that rounds back to 16,777,216, both times. The total never moves.

With Kahan, the first addition also lands on 16,777,216, but the compensation records the lost amount: (16,777,216 − 16,777,216) − 1 = −1. The second term is corrected to 1 − (−1) = 2, and 16,777,216 + 2 = 16,777,218 is exact. The compensation goes back to 0.
Kahan compensated summation
C++
Rust

float sum = 0.0f, c = 0.0f;        // c carries the lost low-order bits
for (float x : values) {
    float y = x - c;               // add back last round's lost part
    float t = sum + y;             // big + small: low bits of y fall off here
    c = (t - sum) - y;             // recover exactly what fell off
    sum = t;
}

Naive summation's error bound grows with the number of terms; Kahan's stays roughly constant, effectively independent of the count until it approaches 1/epsilon (about 8 million terms for a float)[8]. The race below runs the worked example 60,000 times:
Play
Pause
Reset
compile Kahan with -ffast-math
Blue is the naive total and green is Kahan, both plotted as the amount added above the 224 seed; the dashed line is the true sum. The race starts when it scrolls into view and runs 60,000 additions of 1.0. Each naive +1.0 is an exact tie that rounds back to the even neighbor, so the blue line stays flat and ends at 16,777,216 with an error of 60,000. Kahan ends at 16,837,216 with an error of 0; mid-race its error readout flickers between 0 and ±1 because the total itself must be a float. Tick "compile Kahan with -ffast-math" (a GCC and Clang flag that trades IEEE rules for speed) and the race restarts with the compensation deleted, because the optimizer can prove it's "algebraically zero". Kahan collapses onto the naive line. That's a real footgun, covered next.

[Back to IEEE-754 Floating Point](index.md)
