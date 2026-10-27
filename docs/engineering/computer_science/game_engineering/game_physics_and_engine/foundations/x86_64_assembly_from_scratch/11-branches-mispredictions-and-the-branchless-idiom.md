# Branches, Mispredictions, and the Branchless Idiom


A culling loop tests each object against the frustum and skips the invisible ones. On a level where objects are scattered at random, that one if can cost more than the test itself. The reason is branch prediction. The CPU does not wait for a conditional branch to resolve before fetching the next instructions. It predicts the outcome of every branch (using per-branch and global history, an idea refined continuously since the two-level predictors of the early 1990s[18]) and speculatively executes the predicted path. When the prediction is right (the usual case for tight loops, monotonic conditions, and any pattern with a stable history), the branch is approximately free. When it is wrong, the CPU discards the speculative work and refetches; the cost is roughly ten to twenty cycles on recent Intel and AMD cores[12].

"Approximately free" is worth measuring. Cloudflare's 2021 microbenchmark of long chains of predicted unconditional jumps reports about 2 cycles per jmp on an Intel Xeon Gold 6262 (degrading past the ~4096-entry BTB capacity), about 3.5 cycles on Zen 2 (AMD EPYC 7642), rising to about 10.5 once the chain passes 4096 jumps, and 1 cycle on Apple's M1 when the code fits in 4 KB, 3 cycles beyond that. On the EPYC, never-taken conditional branches cost about 0.3 cycles each no matter how many there were[30]. Two consequences. First, a correctly predicted taken branch still costs one to a few cycles of front-end throughput; a correctly predicted not-taken branch is close to free. Second, the cost is non-linear in the amount of hot code: a loop whose taken branches fit in the BTB and one that just overflows it can differ by about 3× with no source change.

For data-dependent branches that the predictor cannot learn (a random check against a 50/50 input, or a comparison against a key that changes per iteration), the misprediction cost dominates. The remedy is branchless code: replace the conditional with a computation that produces the same result regardless of the predicate, using cmovcc or bit tricks. Take the absolute-value-of-int problem:
abs.s · branchy vs branchless

; Branchy: easy to read, mispredicts about half the time on random signs
abs_branchy:
  test edi, edi
  jns  .Lpos                   ; jump if the sign flag is clear (x >= 0)
  neg  edi
.Lpos:
  mov  eax, edi
  ret

; Branchless: same answer, no jump. Cost is constant.
abs_branchless:
  mov  eax, edi                 ; eax = x (the default)
  neg  eax                      ; eax = -x; flags reflect -x
  cmovl eax, edi                 ; if -x < 0 (i.e., x was positive), take the original instead
  ret                            ; eax holds |x|; UB for x = INT_MIN, same as the branchy form

The older bit-trick form, from before cmov existed, is mov eax, edi; cdq; xor eax, edx; sub eax, edx. cdq sign-extends eax into edx, producing an all-ones mask if x is negative and an all-zeros mask otherwise; the xor and sub then compute (x XOR mask) − mask, which evaluates to x when the mask is zero and to −x when the mask is all-ones. Four instructions, no jump, and no flags read[19].

Worked example

x = −5. cdq fills edx with copies of the sign bit: 0xFFFFFFFF, which is −1.

xor eax, edx flips every bit of −5, giving 4. sub eax, edx computes 4 − (−1) = 5.

For x = 5, edx is 0, the xor and sub change nothing, and the result is 5.

When to use which. Branchless is faster when the predictor cannot learn the pattern. It is slower when the predictor can, because cmov introduces an unconditional data dependency: the result waits for both the source and the predicate, even when the predicate could have been predicted and the dependent computation skipped. The standard demonstration is one of the most-upvoted questions on Stack Overflow[20]: summing the elements at or above a threshold in a 32,768-element array runs several times faster when the array is sorted first, because the predictor mispredicts only around the point where the values cross the threshold. The accepted answer's branchless version runs at the same speed on both inputs and, on the sorted one, is slightly slower than the branchy version. (Current compilers at -O3 often emit cmov or vectorize that loop themselves, which erases the difference.)

Worked example

The widget below charges 1 cycle for a correctly predicted branch, 14 for a mispredict, and 3 per element for the branchless version, over 256 elements.

Branchless: 256 × 3 = 768 cycles on any input.

Branchy on sorted input: 3 mispredicts, so 256 + 3 × 13 = 295 cycles. On random input about half the predictions miss: 128 mispredicts would give 256 + 128 × 13 = 1,920 cycles, 2.5 times the branchless cost. Each run draws a new random array, so the widget's count varies around that, roughly 1,750 to 2,050 in repeated runs.

The widget models this. It runs the same predicate over an array that is either random or sorted; the branchy version is fast on the sorted input and slow on the random one:
Live · Branchy vs branchless
random input
sorted input
Run ▶
Reset ⟲
branchy cycles
···
branchless cycles
···
branchy ÷ branchless
···
The two bars fill with modelled cycles as the loop walks the array, and the strip underneath shows the input with a cursor. Pick random or sorted input and press Run: random input lands around 1,750 to 2,050 cycles against the branchless 768 (a new random array each run), sorted input reads 295 against 768 (0.38×). A simplified model. The predictor is a single 2-bit saturating counter. The branchy version pays 14 cycles per mispredict (within the range Agner Fog measures for recent Intel and AMD cores[12]) and 1 cycle per correct prediction; the branchless version pays 3 cycles per element unconditionally. On sorted data the counter mispredicts only at the start and twice at the crossover, and the branchy version wins; on random data it mispredicts about half the time and the branchless version wins.
Check Your Understanding Q1 of 1

Inside a hot rendering loop you have if (object.visible) draw(object); over a million calls per frame, where visible is true for ~95% of objects. A junior changes the if to a branchless dispatch via a function-pointer table: void (*fn[2])(Object&) = {noop, draw}; fn[object.visible](object);. What probably happens?
The new code is faster because branchless is always faster.
The new code is probably slower. The original branch was right about 95% of the time and nearly free; the table version adds a load and an indirect call for every object, including a real call to noop for the invisible ones.
Same speed, both compile to the same code.
0/1 correct

[Back to x86-64 Assembly from Scratch](index.md)
