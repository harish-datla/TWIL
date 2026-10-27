# Benchmarks, contamination and error bars

Model A scores 71% on a benchmark and model B scores 70%. Is A better? Before you can answer, you need to know what the benchmark measures, whether the models saw its questions during training, and how much the score would move on a different set of questions.

Public benchmarks are fixed test sets with a scoring rule. MMLU is multiple-choice questions over 57 subjects, so random guessing scores 25%. GSM8K's test split is 1,319 grade-school word problems, scored on the final number. HumanEval scores generated code by running unit tests on it.[10][11][12] Because these test sets are public, they leak into the web text that models are trained on. The lm-evaluation-harness checks for this with a rule from Appendix C of the GPT-3 paper, with n fixed at 13 (GPT-3 used 8 to 13 depending on the dataset). A test item counts as contaminated when any 13-gram from it, a run of 13 consecutive pieces of text, appears in the training data, and a clean score leaves those items out.[13]

For code, one attempt per problem understates what a model can do, so HumanEval reports pass@k: the probability that at least one of k attempts passes the tests. Generating exactly k samples per problem and checking whether any passes gives a noisy estimate. The HumanEval evaluation code instead draws n ≥ k samples, counts the c that pass, and computes 1 − C(n − c, k)/C(n, k), where C(a, b) counts the ways to choose b items from a. That is the chance that k samples picked from the n include at least one that passed. It skips pass@k when n < k, where no unbiased estimate exists.[12]

Worked example

n = 10 samples, c = 3 pass.

pass@1 = 1 − C(7, 1)/C(10, 1) = 1 − 7/10 = 0.30, the plain pass rate.

pass@5 = 1 − C(7, 5)/C(10, 5) = 1 − 21/252 ≈ 0.92: five tries almost always include one of the three good programs.

Every benchmark score is an average over a finite set of questions, so it has the standard error of an average: a measure of how much it would move if you reran it on a fresh set of questions of the same kind. Treat each question as a coin flip that comes up "correct" with probability p. For a score p over n questions the standard error is √(p(1 − p)/n), and a 95% interval is roughly ±2 standard errors.

Worked example

A model scores 70% on 1,000 questions. Standard error = √(0.7 × 0.3 / 1000) ≈ 0.0145, or 1.45 points. The 95% interval is about ±1.96 × 1.45 ≈ ±2.8 points.

A one-point gap between two models on that benchmark is well inside the noise.
Put an error bar on a benchmark score
Questions n
1000
True accuracy, %
70
Rival is worse by, points
Reset
1000 questions, true accuracy 70%: the measured score has standard error √(p(1−p)/n) = 1.45 points, so a 95% interval spans about ±2.8 points. A second model 2 points worse scores higher than the better one on a fresh set of 1000 questions with probability 16.7% (normal approximation).

The curve is the distribution of scores a model with the chosen true accuracy would get on a fresh set of questions; the green line marks a rival that is a few points worse. Change the number of questions, the true accuracy and the gap. The curve is the exact binomial distribution, computed from the true accuracy and the question count. At 1,000 questions and 70%, a run lands within about ±2.8 points of the truth 95% of the time, and a model that is really 2 points worse outscores the better one on a fresh thousand-question set about 17% of the time. At 100 questions that becomes 38%, close to a coin flip. The calculation treats questions as independent; correlated questions make the real noise larger.
Check Your Understanding Q1 of 2

Model A scores 71.3% and model B scores 70.6% on the same 500-question benchmark. What does that establish?
Little: the standard error at this size is about 2 points, larger than the gap.
A is better, since it scored higher on identical questions.
A is better by exactly 0.7 points on the task.
0/2 correct
