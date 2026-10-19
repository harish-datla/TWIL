# Rollback

In a fighting game a punch can be active for only three frames, so a few frames of delay change which hits land. Waiting for the opponent's input before simulating each frame adds that delay to every frame. Rollback netcode (GGPO, the fighting-game standard) runs a deterministic sim and predicts the remote player's input (assume they keep doing what you last heard, "carry-forward"), simulating forward immediately so the game feels offline-responsive. When the real input arrives and differs, it rolls back to the saved state at that frame and re-simulates forward to the present with the corrected input[12].
Not lag-free, and it demands two things at once

Rollback hides remote latency but doesn't erase it: a misprediction produces a visible correction/teleport. It requires both a fully deterministic sim and the ability to save/restore the entire game state cheaply every frame (ideally one contiguous block you can memcpy). The cost scales with the misprediction window (latency in frames, minus any input delay): at 60 fps with three frames of input delay, supporting 300 ms of one-way latency means re-simulating up to 15 frames inside one 16.6 ms display frame; overrun that budget and the resim becomes its own spiral of death[13]. It predicts the remote input; your own is applied directly. Full-state rollback best fits 2-player P2P with small state. In a 64-player shooter the state to save is large, each resim steps every player, and with 63 remote inputs to guess, some guess is wrong on most frames, so a near-full-window resim runs almost every frame. The idea survives there in narrower form: §5's snap-and-replay reconciliation is a rollback of your own predicted state.

Worked example

At 60 fps, 300 ms of one-way latency is 0.3 × 60 = 18 frames. Three frames of input delay cover 3 of them, leaving a 15-frame window: when a late input turns out to differ from the guess, the game restores the state from 15 frames ago and steps the simulation 15 times, plus the current frame, inside one 16.7 ms display frame. That's about 1 ms per simulation step, before rendering gets any time at all.

The widget predicts the remote input; when a real input arrives that differs, the sim rolls back N frames and re-simulates. Raise latency to grow the rollback window:
Latency 4 frames
Mispredicts 35%
Drag the latency and mispredict sliders and watch the frame timeline and the remote character. Each frame the remote input is predicted and the sim runs ahead. When a real input differs, the timeline rolls back to the saved frame and re-simulates forward (fast but visible), and the remote character pops to its corrected spot. Raising latency widens the window: more frames to re-simulate per correction, and a bigger pop because more frames ran on the wrong input. The faint ring is the remote's actual position, which the local sim only learns about one latency window later; the fading red outline marks where the mispredicted present had been drawn before the correction. Rollbacks still fire with the mispredict slider at zero: every turn at a track edge is an input the predictor could not know in advance, so it mispredicts by construction.
The rollback core (predict remote, roll back, re-simulate)
C++
Rust

constexpr int MAX_ROLLBACK = 8;             // prediction window; a real implementation stalls the sim past this
GameState savedStates[MAX_ROLLBACK];        // ring of per-frame saves (must be cheap to copy)
Input remoteInputs[MAX_ROLLBACK];            // remote input each frame was SIMULATED with (predicted or real)
GameState currentState;                     // the live simulation state
int confirmedFrame = -1;                    // last frame with a REAL remote input
int presentFrame = 0;                       // newest frame advance() has simulated

Input predictRemote() {                       // carry-forward: keep doing what we last heard
    return remoteInputs[(confirmedFrame >= 0 ? confirmedFrame : 0) % MAX_ROLLBACK];
}
void advance(int frame, Input local) {
    presentFrame = frame;
    savedStates[frame % MAX_ROLLBACK] = currentState;    // SAVE before stepping
    Input predicted = predictRemote();
    remoteInputs[frame % MAX_ROLLBACK] = predicted;      // RECORD the guess so onRemoteInput can compare
    simulate(currentState, local, predicted);            // step with the PREDICTED remote input
}
void onRemoteInput(int frame, Input real) {
    if (real == remoteInputs[frame % MAX_ROLLBACK]) { confirmedFrame = frame; return; } // guessed right
    remoteInputs[frame % MAX_ROLLBACK] = real;
    confirmedFrame = frame;                              // move BEFORE resim: carry-forward uses the new input
    currentState = savedStates[frame % MAX_ROLLBACK];    // ROLL BACK to that saved frame
    for (int f = frame; f <= presentFrame; ++f) {        // RE-SIMULATE forward to now
        savedStates[f % MAX_ROLLBACK] = currentState;    // RE-SAVE: a later rollback needs corrected states
        Input remote = (f == frame) ? real : predictRemote();
        remoteInputs[f % MAX_ROLLBACK] = remote;         // re-record what this frame actually ran with
        simulate(currentState, localInputAt(f), remote);
    }                                                    // the re-simulated present renders next: the visible pop
}

