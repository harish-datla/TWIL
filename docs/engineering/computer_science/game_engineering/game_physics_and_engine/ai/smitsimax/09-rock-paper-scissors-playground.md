# Rock-Paper-Scissors playground

Rock-Paper-Scissors is a minimal simultaneous-move game: two players, three actions each, symmetric payoffs. The unique Nash equilibrium is to play each action with equal probability (1/3, 1/3, 1/3). Any deviation from uniform can be exploited by an opponent who shifts weight toward the counter-action.

Worked example

Suppose the agent leans toward Paper: 50% Paper, 25% Rock, 25% Scissors. An opponent who always throws Scissors wins against Paper half the time, loses to Rock a quarter of the time and ties the rest, for an expected +0.50 − 0.25 = +0.25 per round. Against the uniform 1/3 mix, every fixed throw (and every mix) expects exactly 0.

The widget below lets you play RPS against a Smitsimax agent. The agent runs 1,000 iterations of Smitsimax before each round, building a tree of depth 1 (one simultaneous round), with an opponent model added on top of plain self-play. The bar chart on the right shows the agent's current strategy (visit distribution over Rock, Paper, Scissors). Play close to uniform and the distribution stays near 1/3 each. Fall into a pattern and the agent's opponent model, built from your move history, shifts weight toward the counter.
Live · Play RPS vs Smitsimax
reset scores
Rock
Paper
Scissors
your wins
0
ties
0
agent wins
0
rounds
0
Click Rock, Paper or Scissors to play a round; the bars show the agent's visit distribution and the counters keep score. The agent runs a fresh Smitsimax search (1,000 iterations) before each round. Inside the search, the opponent model mixes self-play with the empirical distribution of your last 20 moves, weighted more heavily as history accumulates. With no history to exploit (early rounds), the visit distribution lands near uniform. Once your play shows a lean, the counter-move starts scoring highest in the search and the visits shift toward it: favor Rock and the bars tilt toward Paper.
