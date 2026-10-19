# Information sets

The first optimization most people try is caching the game state in each node, so an iteration can resume from there instead of re-simulating from the root. In Smitsimax that silently breaks the search.

A node in a standard MCTS tree maps to exactly one game state. A node in a Smitsimax tree does not. Consider Player 1's tree at depth 2. On iteration 37, Player 1 selected moves A→B, while Player 2 (in their own tree) selected X→Y. On iteration 52, Player 1 selected the same A→B, but Player 2 selected X→Z this time. Player 1's node for "B" was visited in both iterations, but the underlying game state was different because the opponent made different choices.

This is analogous to an information set in game theory: a set of game states that are indistinguishable from a player's perspective. Player 1 knows they played A→B but doesn't know what Player 2 did, so all game states consistent with Player 1 playing A→B belong to the same information set. The Smitsimax node for "B" aggregates statistics over this entire information set.

This has a consequence: you cannot store state-dependent information in a Smitsimax node. In standard MCTS, you can cache the game state at each node. In Smitsimax, the game state at a node varies between iterations. Every iteration must re-simulate from the root to reconstruct the actual game state at each depth.
Critical constraint

Smitsimax requires that each player's legal moves at a given depth do not depend on what the other players chose earlier in the iteration. If Player 2 choosing X on turn 1 can make Player 1's move C illegal on turn 2 (it stays legal when Player 2 chooses Y), the tree is invalid: node C is a child in Player 1's tree, but the simulation breaks on iterations where Player 2 happened to pick X. MSmits notes that this rules the method out for many games; it holds in Coders Strike Back, where a pod can always thrust, steer and shield, and in Code of Kutulu[9]. A common workaround is to keep a fixed move list and map a move that turns out to be illegal to a fallback such as "wait".
