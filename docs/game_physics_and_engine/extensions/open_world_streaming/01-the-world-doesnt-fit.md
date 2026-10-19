# The world doesn't fit

The player can ride to any mountain on the horizon without a loading screen, but the whole world will never fit in memory at once. A modern open world is tens to hundreds of gigabytes on disk; runtime memory is a fraction of that. The streamer's job is to keep resident only the slice of the world near the camera, paging cells in as the player approaches and out as they recede. Mark Cerny's PS5 talk framed the change in exactly these terms: on PS4, system memory had to hold everything the next 30 seconds or so of gameplay might use, while a fast enough SSD lets a game load data when it turns out to be needed instead of parking it in RAM in advance[1].

Worked example

Say a 16 × 16 km world is cut into 256 m squares: 16,000 / 256 = 62.5 cells a side, about 3,900 cells in all. Load everything within 768 m of the player and the circle covers π × 768² / 256² ≈ 28 cells, under 1% of the world. The streamer's whole job is keeping the right 28 resident as the player moves.
Two budgets, evicted independently

Keep RAM and VRAM separate. Gameplay data, audio, navigation, and collision live in system memory; meshes and textures are uploaded to GPU memory. Each pool has its own budget and its own eviction policy, so a cell can be RAM-resident while its textures are still streaming into VRAM. This tutorial sits above the I/O streamer and the residency pool you built in File Streaming; it decides which cells to ask that streamer for.
