# An agent learns from scores, not answers

Pretraining teaches a language model by showing it text and asking for the next word, so every example comes with the right answer. Many problems don't come with answers. A chess program finds out whether a move was good only when the game ends, many moves later. A chatbot gets a score after it replies; nobody hands it a correct reply to copy. And the move an agent makes decides which positions it sees next, so its own choices shape its training data. Reinforcement learning (RL) is the family of methods for this setting: an agent takes actions, gets back numeric rewards, and learns to pick actions that collect more reward over time. The agent maximizes the reward you wrote down, which isn't always the behavior you meant: it can raise its score by doing something you didn't intend.

To reason about this, RL describes the world as a Markov decision process (MDP). An MDP has states (where things stand right now), actions, and rules that say which next state and reward can follow each state and action.[1]

The agent's rule for choosing an action in each state is its policy. The "Markov" part is a demand on the state: it must hold everything needed to predict what comes next, so the earlier history adds nothing. A single screenshot of a ball game can fail that test, because it doesn't show which way the ball is moving.[1]

Every demo on this page uses one tiny world. It's a 4×4 grid with squares numbered 0 to 15, left to right along each row, starting at the top left. Square 5 is a wall. The agent can try to move up, right, down or left; walking into the edge or the wall leaves it where it was. Entering square 15, the goal, earns +1 and ends the episode (one complete run from start to finish). Every other move earns −0.04, a small cost that makes shorter routes better. The rules are fully known and nothing is random, which is what makes every number on this page checkable. Real games add hidden information and randomness that this grid leaves out.

The agent's goal is the total reward it collects from now on. Rewards further in the future usually count for less: each extra step multiplies a reward by a discount factor γ (gamma) between 0 and 1. That discounted total is the return.

Worked example

From square 6 (second row, third column) the shortest route to the goal is right to 7, down to 11, down to 15: three moves that earn −0.04, −0.04 and +1.

With γ = 0.9: return = −0.04 + 0.9 × (−0.04) + 0.9² × 1 = −0.04 − 0.036 + 0.81 = 0.734.

The general formula adds up every future reward with its discount, which lets an agent compare a route that pays off soon with one that pays off later.
G_t = reward_(t+1) + γ × reward_(t+2) + γ² × reward_(t+3) + …

In words: the return from time step t, Gt, is the next reward, plus γ times the reward after that, plus γ² times the one after that, and so on. Each step further away shrinks a reward by another factor of γ. In an episode, the sum stops at the move that ends it.

The return depends on which moves the agent makes, so a value is always tied to a policy. A state's value V(s) is the return you expect starting from state s and following that policy. An action value Q(s, a) is the same thing with the first action fixed to a. In the worked example, square 6's value under the shortest-route policy is 0.734.

A value isn't a probability of success. Its units come from the rewards, and it can be negative. Multiply every reward by 10 and every value grows tenfold, while the best policy stays the same.[1]
