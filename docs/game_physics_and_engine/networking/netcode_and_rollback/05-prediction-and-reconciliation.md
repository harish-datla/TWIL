# Prediction & reconciliation

You press forward in a shooter with a 120 ms round trip to the server. If your character waits for the server's answer, it starts moving 120 ms after your finger did, which feels like steering through mud. Under an authoritative server, waiting for the round-trip means input lag of at least the full RTT (server processing and tick alignment add more). Client-side prediction applies your own input immediately to your local copy, tagging each input with a sequence number and storing it in a pending buffer[7]. Each server update then describes a moment the client has already predicted past, so the two disagree until the client reconciles.
Reconciliation replays; it does not just snap

The server's state update carries the sequence number of the last input it processed. On receipt the client: (1) snaps to the authoritative state, (2) discards pending inputs up to that ack, (3) replays the still-unacknowledged inputs on top. Replaying is what keeps your predicted position correct when the server agrees. A correction/rubber-band appears only when it genuinely disagreed (a misprediction, e.g. you got shoved); some engines blend that correction over a short window instead of popping, as Source does with cl_smoothtime[9]. Prediction makes your character responsive; it does nothing for other players (that's interpolation, §6).

Worked example

At a 120 ms round trip and one input per 60 Hz frame, about 0.12 × 60 ≈ 7 inputs are in flight at any moment. A server update arrives acking input 500, and the client has inputs 501 to 507 pending. It snaps to the server's position for input 500, drops 500 and everything older from its buffer, and re-applies 501 to 507. If the server agreed, that lands exactly where the client already was and nothing moves. If another player shoved you at input 498, the replay starts from the shoved position and you see the correction.

The character follows a moving input; raise the latency. With no prediction it lags behind your input by the full round trip. Prediction with no reconciliation feels instant but never corrects its mispredictions, so it drifts off the authoritative position and never recovers. Reconciliation is the half that makes prediction usable: it snaps the character back onto authority and replays your unacked inputs:
no prediction
prediction
prediction + reconciliation
Round trip 120ms
Pick a mode and drag the round-trip slider. The blue ring is where the server says you actually are; the green dot is what your client draws; the colored bar between them is the prediction error. No prediction: the character renders the late server state, so it trails your input by the full round trip (the demo lumps both legs of the trip into the snapshot's return path). Prediction only: your own input is applied instantly, but the server keeps applying input the client could not predict (another player shoved you, or the server resolved a collision you had not seen), and with no reconciliation that error accumulates: the green dot drifts off the blue ring and never recovers. Prediction + reconciliation: same instant response, but every server update the client snaps to authority and replays your unacked inputs, so the error collapses back toward zero. The snap is invisible when the server agreed and pops as a rubber-band only when it genuinely disagreed. The input buffer is keyed by timestamp here; the code panel below keys the same snap-and-replay by sequence number.
Client prediction + reconciliation (replay, not snap)
C++
Rust

#include <cstdint>
#include <deque>

std::deque<Input> pendingInputs;          // unacknowledged inputs, in order
uint32_t inputSequence = 0;
State predictedState;                       // what we render locally

void onLocalInput(float dx, float dt) {
    Input input{ ++inputSequence, dx, dt };
    applyInput(predictedState, input);      // PREDICT: apply now, don't wait for the server
    sendToServer(input);                    // the server will ack it by sequence number
    pendingInputs.push_back(input);          // keep it until acked
}
void onServerState(const State& serverState, uint32_t lastProcessedInput) {
    predictedState = serverState;            // RECONCILE 1: snap to authority
    while (!pendingInputs.empty() && pendingInputs.front().sequence <= lastProcessedInput)
        pendingInputs.pop_front();           // drop what the server already ran
    for (const Input& input : pendingInputs)
        applyInput(predictedState, input);  // RECONCILE 2: REPLAY unacked inputs
}

