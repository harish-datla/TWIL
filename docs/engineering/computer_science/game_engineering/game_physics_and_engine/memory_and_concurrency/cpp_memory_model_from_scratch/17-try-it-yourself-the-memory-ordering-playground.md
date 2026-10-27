# Try It Yourself: The Memory-Ordering Playground


You're about to weaken an ordering in shipping code, and you want to know what it lets through on each CPU before a player finds out. The playground below runs the four litmus tests from §4 and §5 against models of x86-TSO, ARMv8, and SC. Edit the ordering in brackets on any STORE or LOAD line, press run, and the playground samples 10,000 executions and counts how often the invariant breaks. Each model maps the orderings to that architecture's standard instructions (on x86, only a seq_cst store drains the store buffer; on ARMv8, acquire and seq_cst loads become LDAR and release and seq_cst stores become STLR) and allows exactly the reorderings that hardware allows.
⌬ Ordering Playground
Run 10,000 trials ▶
// Message Passing pattern
// Producer writes data, then sets flag.
// Consumer reads flag, then reads data.
// Invariant: if consumer sees flag=1, it must see data=42.

INIT: data=0, flag=0

THREAD producer:
  STORE data = 42 [relaxed]
  STORE flag = 1  [release]    // try changing to [relaxed]

THREAD consumer:
  r0 = LOAD flag  [acquire]    // try changing to [relaxed]
  IF r0 == 1:
    r1 = LOAD data [relaxed]

INVARIANT: r0 == 1 IMPLIES r1 == 42
Preset loaded. Click Run to simulate.

Worked example

Keep the Message Passing preset and change the flag store from [release] to [relaxed].

Under x86-TSO, the run reports 0 / 10,000 violations, with a warning that the C++ standard still allows the failing outcome: x86 doesn't reorder the two stores, but the compiler may.

Switch to ARMv8 and run again: violations appear, and the output names the failing outcome, r0=1, r1=0 (flag seen, payload not). Put [release] back and the ARMv8 run drops to 0.

The allowed-or-forbidden verdicts follow the hardware models; the failure rates are made up. A real chip might produce an allowed outcome once in a billion runs or never (LB is architecturally allowed on ARMv8 but rarely observed), so read the output as which orderings let the invariant fail. Two things the models leave out: the compiler, which may reorder relaxed accesses even on x86, and the C++ standard's own rules, which are weaker than either chip. The clearest case is SB with release stores and acquire loads: x86 lets (0, 0) happen, ARMv8 forbids it because an LDAR can't pass an earlier STLR, and C++ allows it, so portable code needs seq_cst there regardless of what one chip does. The same goes for IRIW: ARMv8 forbids it once the reads are acquire, but POWER doesn't, and neither does the C++ standard.

[Back to The C++ Memory Model from Scratch](index.md)
