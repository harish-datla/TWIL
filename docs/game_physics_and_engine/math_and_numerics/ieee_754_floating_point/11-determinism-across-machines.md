# Determinism across Machines


In a lockstep game, every machine runs the same simulation from the same player inputs and sends only the inputs over the network. A real-time strategy (RTS) game uses it because thousands of units would be too much state to send. It only works if every machine computes bit-identical results, and identical source can produce different float results on different machines. That is why a lockstep RTS or a fighting-game replay can desync on a teammate's PC, and why the Game Loop tutorial, next in the series, needs more than a fixed timestep to make a simulation deterministic.

Worked example

Run the line r = a*b + c from §9 (a = b = 1.000244140625, c = −1) on three builds:

    x86-64 with SSE and contraction off: 0.00048828125.
    The same source contracted into an FMA: 0.000488340854644775390625.
    A 32-bit x87 build that keeps the product in an 80-bit register: the product is exact there, so it also gets 0.000488340854644775390625.

The results differ by 1,024 ULP. A lockstep game that checksums its state every frame flags a desync on the first frame this line runs.

The causes are specific; floats are not random[11]:

    Different compilers and optimization levels reassociate and contract differently.
    FMA contraction on one platform but not another.
    x87 80-bit intermediates vs SSE 32/64-bit on x86. x87 is the original x86 floating-point unit, which keeps intermediates at 80 bits by default; SSE rounds every result to exactly float or double.
    Transcendental functions. IEEE-754 requires + − × ÷ sqrt fma and remainder to be correctly rounded, but it does not require it of sin, cos, exp, so vendor math libraries legitimately differ[11].
    FTZ/DAZ mode differences (next section).

"Use doubles" does not fix determinism

More bits doesn't make two different rounding sequences agree. Cross-platform determinism comes from controlling the operations: fixed-point arithmetic, or strict same-compiler, same-flags builds with contraction and x87 disabled and transcendentals replaced by your own implementations. Glenn Fiedler's writeup quotes a developer of a shipped lockstep game (Battlezone II) whose team wrapped sin, cos, tan and their inverses in non-optimized calls to keep them at single precision, because AMD and Intel processors returned slightly different results for them[11].

[Back to IEEE-754 Floating Point](index.md)
