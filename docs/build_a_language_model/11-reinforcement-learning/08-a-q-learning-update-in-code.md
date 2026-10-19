# A Q-learning update in code

These complete programs implement the Q-learning update from section 03 for one move. The caller passes the next state's Q values and a flag that's true only when the task really ended. The tests use the numbers from section 06 (reward 1, best next value 5, γ = 0.9): they check that a truncated move keeps its future term and gets a target of 5.5, while a terminated one gets 1. They also check that the target is computed before the current row changes. That matters when a move bumps into a wall and the agent stays put, so the current and next rows are the same. The C++ test passes the same array twice; Rust's borrowing rules forbid that, so the Rust test passes a copy.
A checked Q-learning update, complete programs
C++20
Rust 2021

#include <array>
#include <algorithm>
#include <cassert>
#include <cmath>
using Actions = std::array<double,4>;
double update(Actions& current, int action, double reward,
              const Actions& next, double discount, double learningRate,
              bool terminated) {
    assert(action >= 0 && action < 4);
    assert(discount >= 0 && discount <= 1);
    assert(learningRate > 0 && learningRate <= 1);
    // next refers to the final observation, not an auto-reset observation.
    const double bootstrap = terminated ? 0.0 : *std::max_element(next.begin(),next.end());
    const double target = reward + discount*bootstrap;
    const double error = target-current[action];
    current[action] += learningRate*error; // Read the whole target before modifying Q.
    return error;
}
int main() {
    Actions current{0,0,0,0};
    const Actions next{2,5,1,0};
    const double error = update(current,1,1,next,0.9,0.4,false);
    assert(std::abs(error-5.5) < 1e-12);
    assert(std::abs(current[1]-2.2) < 1e-12);
    Actions terminal{0,0,0,0};
    update(terminal,1,1,next,0.9,0.4,true);
    assert(std::abs(terminal[1]-0.4) < 1e-12);
    // An external time limit has terminated=false and retains bootstrapping.
    Actions truncated{0,0,0,0};
    update(truncated,1,1,next,0.9,0.4,false);
    assert(truncated == current);
    // Aliased current/next is safe because the target is evaluated first.
    Actions self{1,2,3,4};
    update(self,0,0,self,0.9,1.0,false);
    assert(std::abs(self[0]-3.6) < 1e-12);
}

The listing leaves out the loop that runs the environment, storage for past experience, schedules for α and ε, and evaluation. It also relies on a table, which stops working once there are too many states to list. Bigger problems replace the table with a neural network that predicts Q values. The network shares weights across states, so an update for one state also shifts the predictions for others, which makes training less stable. DQN (deep Q-network), from Mnih et al., added two fixes. A replay buffer stores past moves and trains on random batches drawn from it, which breaks up runs of nearly identical consecutive moves. A separate target network, a copy of the Q network that's refreshed only every so often, computes the targets, so they don't shift with every update.[7]

Neither fix guarantees that deep Q-learning converges, and both add choices of their own: how to sample from the buffer, how old its data may get, how terminal moves are stored, and how often to refresh the target. Test the table version first. Then check the network on small worlds whose true values you can compute, before blaming a failure on the network being too small.
Check Your Understanding Q1 of 2

Why can a baseline change the variance without changing the average gradient?
It removes all randomness from the policy.
It changes the rewards that define the objective.
Its average contribution is the baseline times the gradient of the total probability, and that total is always 1.
0/2 correct
