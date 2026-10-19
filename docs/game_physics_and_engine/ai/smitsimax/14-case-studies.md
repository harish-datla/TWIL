# Case studies

Where the method has been used for real: three CodinGame contests, the general-game-playing program that got there first, and the card game researchers use to test it.
1 · Code of Kutulu (CodinGame, 2018)

A four-player simultaneous-move survival game set in a Lovecraftian maze: each player steers one explorer, avoiding game-controlled monsters while a sanity meter drains. The four-way joint move space makes conventional search hard. MSmits first thought of Smitsimax during this contest and placed 4th[9]. Move legality doesn't depend on the other players, which is what lets the decoupled trees work (§8).
2 · Coders Strike Back (CodinGame, ongoing)

A pod-racing game with two pods per player (four pods total). Each pod has a continuous action space (target angle + thrust), discretized to a manageable set. MSmits reached the top 10 in about 20 hours of coding, using Smitsimax with four trees (one per pod)[9]. His blocker pod's evaluation rewards reducing the opponents' progress and also rewards his own runner's progress. With those shared goals, the blocker sometimes learned to use its shield to boost the runner ahead, a cooperative move nobody coded explicitly.
3 · SnakeByte (CodinGame Winter Challenge, 2026)

A multi-snake simultaneous-move game. The first-place solution by "frictionless" used Smitsimax with one tree per snake (merging nodes whose snake head position and direction match) and a maximum depth of 12, simulating around 50,000 turns per 50 ms. The fifth-place solution by "sZoom" used Smitsimax with 4,000–6,000 iterations and roughly 250,000 simulations per 45 ms, growing the depth as the search ran. Both describe the one-tree-per-snake structure in their postmortems[21].
4 · CadiaPlayer (GGP, 2007–2008)

Björnsson and Finnsson's CadiaPlayer[3] used decoupled per-player statistics (DUCT) for simultaneous-move games in the AAAI General Game Playing competition, winning in both 2007 and 2008. It predates the Smitsimax name by about a decade and is the academic antecedent MSmits acknowledged.
5 · Goofspiel (academic benchmark)

Goofspiel (the Game of Pure Strategy) is a standard benchmark for SM-MCTS research[7]. Both players simultaneously bid cards to win prize cards, and good play requires mixing. It is the counterpoint to CadiaPlayer: with 1 second per move, Lanctot et al. found regret matching and Online Outcome Sampling beat every other variant, while DUCT playing its most-visited move lost to all of them and produced the most exploitable strategies. DUCT choosing its final move in proportion to visit counts did much better on exploitability.
