# UCB1 and selection

Every node in every tree faces the same choice on each visit: replay the move that has scored best so far, or try one that has barely been sampled and might be better. Replay too eagerly and the search locks onto a move that got lucky early; explore too much and it wastes the budget on bad moves.

Selection within each tree uses the UCB1 formula, the same bandit algorithm that drives standard MCTS. It gives each child a score, its average result plus a bonus that is large when the child has few visits, and picks the child with the highest score. For a child node i with parent p:
UCB1(i) = X̄i + C · √(ln Np / ni)

In words: a child's score is its average reward so far plus an exploration bonus. The bonus grows slowly as the parent is visited more and shrinks as this child is visited more, so a neglected child's score creeps up until it gets picked again.

The first term is exploitation: favor children with high average reward. The second term is exploration: favor children that have been visited less often relative to the parent. The constant C balances the two. In the original UCB1 paper[1], C = √2 (for rewards in [0, 1]) gives a theoretical regret bound. In practice C is tuned per game.

Worked example

Take the widget's parent with 100 visits, so ln 100 ≈ 4.61, and C = 0.5. The child with 50 visits and average 0.75 scores 0.75 + 0.5 × √(4.61 / 50) ≈ 0.75 + 0.15 = 0.90. The child with 12 visits and average 0.65 scores 0.65 + 0.5 × √(4.61 / 12) ≈ 0.65 + 0.31 = 0.96. The child with 4 visits and average 0.30 scores 0.30 + 0.5 × √(4.61 / 4) ≈ 0.30 + 0.54 = 0.84. The 12-visit child wins: a slightly worse average, but enough uncertainty left to be worth another look.

Smitsimax adds a common modification: score scaling. Scores in one game might run from 0 to 10,000 while Rock-Paper-Scissors pays −1, 0 or 1, and one value of C can't balance exploration against both. So the exploitation term is divided by the observed score range:
UCB1(i) = X̄i / (Smax − Smin) + C · √(ln Np / ni)

In words: the same score, with the average divided by the spread between the best and worst results this tree has seen, so the exploitation term no longer depends on the units the game scores in.

MSmits describes this as normalizing to [0, 1][9], which holds when scores start at zero. In general it only bounds the spread: child averages lie inside [Smin, Smax], so after dividing by the range, the gap between any two children's exploitation terms is at most 1. A true [0, 1] mapping would subtract the minimum first: (X̄i − Smin) / (Smax − Smin). The spread is what matters for tuning, though: it puts exploitation differences on the same scale as the exploration term, so one C value behaves consistently across games.

The widget below lets you adjust C and watch how the selection policy shifts. Low C produces greedy, exploitative play. High C spreads visits more evenly, finding moves that are underexplored but potentially strong.
Live · UCB1 exploration vs exploitation
exploration constant C = 1.41
parent visits = 100
Drag C and the parent-visit count and watch which bar gets the gold outline. Five children with different visit counts and average rewards. The purple portion of each bar is the exploitation term (average reward); the green portion is the exploration bonus. The child with the tallest total bar is selected (gold outline). At C = 0, selection is pure greedy and picks the n=50 child with the best average. With 100 parent visits, the less-visited n=12 child takes over around C ≈ 0.3, and above C ≈ 0.8 exploration dominates and the least-visited n=4 child wins despite the worst average. More parent visits raise every bonus and lower both thresholds.

One practical detail from MSmits's implementation[9]: the first 10 visits to a node select children at random rather than by UCB1. His reason is what he calls resonance: early in the search every agent plays badly, a bad move can look good against the other agents' equally bad moves, and without forced randomness all the trees can settle on bad moves together. The random phase also keeps early decisions from resting on one or two noisy samples.
