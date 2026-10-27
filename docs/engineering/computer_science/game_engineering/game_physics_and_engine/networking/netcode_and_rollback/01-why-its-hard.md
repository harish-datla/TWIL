# Why it's hard

Two players' machines need to agree on one game, but every message between them takes time and some never arrive. Netcode works against five constraints: latency (the delay before a packet arrives), jitter (variance in latency), packet loss, bandwidth, and cheating. Latency has a floor you cannot beat: light in fiber travels at roughly 200,000 km/s, so New York to London (~5,500 km great-circle) is about 55 ms round-trip of pure propagation, before cable detours, routing, or queuing add more. The network feeds each peer the inputs or state for the fixed tick it's about to run.

Worked example

New York to London is about 5,500 km. At 200,000 km/s in fiber, one way takes 5,500 / 200,000 = 0.0275 s, so 27.5 ms, and the round trip 55 ms. At 60 frames per second a frame lasts 16.7 ms, so even a perfect network puts the reply 3.3 frames after the press. Every technique on this page is a way of not making the player feel those frames.
