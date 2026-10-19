# Loading radius & priority

A dozen cells are queued and the loader can fetch a few a second. Which goes first? The set of cells to keep resident is a streaming volume around each streaming source (usually the camera, sometimes also a fast vehicle or a spectated player). Within that volume, requests are ordered by priority rather than issued all at once. A common priority blends three signals.

    Distance. Nearer cells first; they are what the player reaches soonest.
    Visibility. Cells inside the view frustum outrank cells behind the camera. Unreal's texture streamer uses the same family of heuristic, ranking by the on-screen texel-to-pixel ratio and whether a thing is visible at all[4].
    Velocity prediction. Bias the volume along the predicted path so a cell finishes loading before the player arrives, rather than as they cross into it.

Worked example

Say the score is distance in meters, doubled for cells outside the view frustum, and lower loads first. Cell A is 300 m away behind the camera and scores 600; cell B is 500 m away straight ahead and scores 500, so B loads first even though it's farther. Prediction moves the volume itself: the §2 widget centers its ring 0.6 s ahead of travel, which at its top speed of 2.2 cells a second puts the center 1.3 cells in front of the player.
Prediction has a failure mode worth seeing

Predicting ahead of travel only helps while the prediction holds. On a sharp turn the velocity is stale, so a cell finishes loading where the player was about to be instead of where they went. In the §2 widget that shows up after the player bounces off the map edge: cells that were loading ahead of the old heading land behind the new one. Predicting harder makes this worse. The fix is a load radius large enough that the unpredicted neighbors are resident anyway, and cheap eviction for the mispredicted cells.
