# Sequential Consistency, the Strawman


Ask most programmers how two threads run and they'll describe taking turns: a few instructions from one thread, then a few from the other, each thread's own steps in the order it wrote them. That picture has a name, sequential consistency (SC), and it's Lamport's 1979 model[4]: every operation, by every thread, can be placed in one global total order that every thread agrees on. Each thread's operations appear in that order in program order; operations from different threads can interleave in any way. The two rules:

    One total order exists.
    Each thread's contributions to that order match its program order.

Worked example

Take §1's program under SC. Thread A runs A1: x = 1, then A2: r0 = y. Thread B runs B1: y = 1, then B2: r1 = x. Keeping each thread's two steps in order leaves six possible sequences:

A1 A2 B1 B2 gives (r0, r1) = (0, 1).
B1 B2 A1 A2 gives (1, 0).
A1 B1 A2 B2, A1 B1 B2 A2, B1 A1 A2 B2 and B1 A1 B2 A2 all give (1, 1).

None gives (0, 0): whichever load runs last comes after both stores. That's why §1's (0, 0) result rules out SC on x86.

SC is what most programmers imagine. No mainstream CPU architecture in use today provides it by default, because SC forbids the store buffer's main trick: a core would have to wait for each store to become visible to every other core before letting a later load complete. Current hardware uses TSO (x86, SPARC, IBM Z) or a weaker model (ARMv8, POWER, RISC-V's RVWMO), and language standards expose those models with knobs to recover SC where you need it.

The SC mental model fails on real hardware in three classic patterns, all of them litmus tests (tiny programs plus a question about one final outcome) from the Sewell and Maranget papers[2][9]:
Store Buffer (SB)

init: x=0, y=0

T0:           T1:
  x = 1;        y = 1;
  r0 = y;       r1 = x;

forbid: r0=0 AND r1=0  (under SC)
allowed: r0=0 AND r1=0  (under TSO and weaker)

Message Passing (MP)

init: data=0, ready=0

T0:                  T1:
  data = 42;           while (ready == 0);
  ready = 1;           r = data;

forbid:  r=0  (under SC and x86-TSO,
              or with release/acquire)
allowed: r=0  (on ARM and POWER with
              plain loads and stores)

Independent Reads of Independent Writes (IRIW)

init: x=0, y=0

T0: x = 1;          T1: y = 1;
T2: r0=x; r1=y;     T3: r2=y; r3=x;

forbid: r0=1, r1=0, r2=1, r3=0  (under SC)
With each reader's two loads kept
in order (dependency or acquire):
  allowed on POWER and ARMv7
  forbidden on x86 (multicopy atomic)
  forbidden on ARMv8 since its
  multicopy-atomic revision

SB is §1's program. MP is the publish pattern from the top of the page: write the data, then set a flag. IRIW shows that without multicopy atomicity there's no single "now" that every core agrees on. Two writes happen in parallel. Two readers, each reading both variables in order, can disagree about the order in which the writes became visible: reader T2 saw x become 1 before y, and reader T3 saw y before x. On x86 this can't happen because TSO is multicopy atomic: a store becomes visible to all other cores at the same moment, even though each core can reorder its own store with its own later load. On POWER and ARMv7 it can. ARMv8 was revised to be multicopy atomic, which no production ARMv8 core had ever violated[35], so IRIW with ordered reads is forbidden on AArch64. With plain loads it still shows up there, because ARMv8 can reorder the two loads themselves, and the SB, MP, and LB outcomes remain allowed.

[Back to The C++ Memory Model from Scratch](index.md)
