# Hot reload

You're tuning the fog in a shader: change a number, rebuild, relaunch, walk back to the spot, look, repeat. Hot reload, swapping the edited file into the running game, closes that loop: a file-watcher detects an edit, the asset is re-cooked and re-read, and the live handle is swapped (the Asset Pipeline owns the cook/swap machinery). For a renderer, shader hot-reload (recompile to SPIR-V, rebuild the pipeline object, swap it in) is often the highest-value reload: you tune a shader and see the result without restarting.

Worked example

Say you make 40 shader tweaks in an afternoon, and each one costs a 90-second rebuild and relaunch plus 30 seconds to get back to the spot: 40 × 2 minutes = 80 minutes of waiting. If a shader reload takes about 3 seconds, the same 40 tweaks cost 40 × 3 = 120 seconds, 2 minutes.
An iteration tool, with edge cases

Hot reload is a dev-build feature, not free or transparent: in-flight GPU work may still reference the old resource (swap behind a fence), layout/format changes invalidate descriptors, and serialized state can stop matching the new code.
Check Your UnderstandingQ1 of 4

A teammate argues you should switch the tools UI from immediate-mode (Dear ImGui) to retained-mode because immediate-mode is slower. What is the accurate framing?
Immediate vs retained is about where UI state lives, not raw speed; immediate-mode rebuilds geometry each frame (a cost) but its win is iteration speed and no state-sync bugs, which is why it is common for debug/tools UI.
Retained-mode has no state to synchronize, so it is strictly simpler.
Immediate-mode retains nothing internally, so it cannot cache anything.
0/4 correct

Wrong answers, and why: immediate vs retained is about where state lives (retained has the sync burden; immediate still caches internally); CPU timers can't measure async GPU work (use timestamp queries read back later, not a same-frame WAIT stall); an average FPS hides hitches that P99 catches; and total time includes callees, so look for self time.
