# Why a Game Engine Needs a Streamer


The PS5 ships with 16 GB of unified memory. Marvel's Spider-Man 2 is around 100 GB on disk. Cyberpunk 2077 is 70 GB. Call of Duty's installed footprint regularly crosses 200 GB. That's the central problem: the world the player walks through is five to ten times larger than the RAM it has to live in. A streamer is the system that hides that fact. It pages assets in as the player approaches them, pages them out when they're no longer visible, and does it fast enough that the player never sees the seams.

A job system (see Job Systems) is about using every CPU core. The streamer is about using every byte of storage bandwidth without blowing the frame budget. Any engine that ships large worlds has to solve it, and the player notices when it's wrong. "Loading…" screens, texture pop-in, geometry pop-in, audio that cuts in two seconds late: streaming is a common cause of all four. The streamer this page builds is about 250 lines in C++ and in Rust, and a JavaScript port runs live in a playground near the end, where you can drive a player through a world and tune cache size, prefetch radius and bandwidth.
The fixed-budget reality

Three constraints make this hard.

    The world is far bigger than RAM. A modern open-world game maintains a hundred gigabytes of textures, meshes, audio, animation, navigation data, and AI scripts. Eight to sixteen of those gigabytes are available at runtime; the rest live on storage and have to come in on demand.
    Storage is far slower than RAM. A read from L1 cache takes around 4 cycles. A read from RAM takes around 300. A read from an NVMe SSD, the flash drive on the fast PCIe bus (the CPU's high-speed link to SSDs and GPUs) that current consoles and gaming PCs use, takes 200,000 to 300,000. The streamer's whole job is to schedule those reads so the player is never waiting for one.
    At 60 fps the frame budget is 16.7 ms. Whatever the streamer does (block, allocate, decompress, copy, bind) has to fit in the slack alongside rendering, physics, audio, and gameplay. Hitches in the streamer are visible as hitches in the game.

The widget below shows the central tradeoff. The first mode loads a whole zone before the player can enter it, the level-based design most PS1-era games shipped with. The second pages content in as the player moves through the world. The level load pays in wait time and in peak memory, and once a zone outgrows the RAM budget it can't work at all.

Worked example

The widget's defaults: an 80 GB world cut into ten zones, an 8 GB RAM budget, and a drive that delivers 2 GB/s. A level load reads the next 8 GB zone before play resumes: 8 ÷ 2 = 4 seconds on a loading screen, with all 8 GB resident. The streaming model keeps a working set of about 4 GB around the player (5% of the world in this model) and only ever waits for the first few nearby assets. Grow the world to 100 GB and each zone becomes 10 GB, more than the 8 GB budget, so the level-load design can't run at all; the streamer's working set grows to 5 GB and still fits.
Live · Level Load vs Streaming
level load
streaming
world size (GB) 80
RAM budget (GB) 8
PLAY ▶
Reset ⟲
wait time
4.0 s
peak RAM
8.0 GB
The strip is a session timeline and the bar below it is RAM in use. Press Play in each mode, then drag the world size past ten times the RAM budget. The model splits the world into ten zones and loads at 2 GB/s: at the defaults, level load reads "4.0 s" of wait and "8.0 GB" of peak RAM, and streaming reads "100 ms" and "4.0 GB". A level-load design has to fit the whole zone in RAM, so at 100 GB with an 8 GB budget it reads "won't fit". The streaming design only pays for what's near the player.
Check Your Understanding Q1 of 2

Your open-world game has a 100 GB installation. The console has 16 GB of RAM. How does the game run at all?
Most of the installation is duplicate or rarely-touched data, so the live game only ever needs a few gigabytes of it.
A streamer pages assets in as the player approaches them and evicts them when they go out of view, so the resident working set fits in RAM even though the total content does not.
The OS’s virtual memory swaps cold allocations out to a paging file as the game touches new areas.
0/2 correct

[Back to File Streaming for Game Engines](index.md)
