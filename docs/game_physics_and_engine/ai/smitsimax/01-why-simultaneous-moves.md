# Why simultaneous moves

Many games aren't turn-based. In StarCraft, both players issue commands at the same time. In Pokémon, both trainers pick a move before either executes. In competitive Rocket League, all six cars accelerate, boost, and jump on the same tick. Card games like poker have a related problem: hidden cards mean you choose without knowing what your opponent holds.

The game-theory term is simultaneous-move game: everyone chooses without seeing anyone else's choice. The classic example is Rock-Paper-Scissors: both players reveal at the same time. Nobody goes second, and no single move is "best". The only stable strategy pair is a Nash equilibrium, a pair of strategies where neither player gains by changing theirs alone: each player randomizes, 1/3 on each action.

Minimax, the classic search for turn-based games, assumes you move and then your opponent answers with the move that's worst for you. That doesn't apply directly here: the second mover gets to see the first move, an information advantage nobody has in a simultaneous game. Standard MCTS has the same problem, as §4 shows. SMITSIMAX is one practical answer: a variant of MCTS for games where everyone acts at once.
Check Your Understanding Q1 of 1

Two players each steer a unit on a grid. Every turn both pick one of 20 moves at the same time, and a game lasts 50 turns. You have 50 ms per turn to choose a move. Which approach fits best?
Depth-first minimax with alpha-beta pruning, with the players taking turns in the tree.
Monte Carlo tree search with independent selection per player.
Solve the full game exactly by enumerating every joint move sequence.
0/1 correct
