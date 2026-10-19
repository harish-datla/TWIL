# The integration budget

The game holds 60 fps on an NVMe SSD, yet it stutters for one frame every time the player crosses into a new region. The load-bearing idea of the whole tutorial: a streaming hitch (one frame that blows the budget) is usually not the disk read. On an SSD the bytes are already in RAM by the time you notice. A common cause is the main-thread cost of making the loaded data live: registering components, building GPU resources, adding primitives to the scene, creating physics bodies. In Unreal, making a streamed level visible runs AddToWorld, which applies the level transform, updates and registers components, initializes actors and routes their initialization functions[9]; Epic's level-streaming hitching guide covers how to spot these hitches in Unreal Insights captures[7]. Unity's Addressables documentation says the same of scene loads: the load is async, but "the final part of scene loading requires operation on the main thread. This can be blocking."[8]
The fix is to time-slice the integration, not to buy a faster disk

Give the per-frame activation work a time budget and stop when it's spent, finishing the cell over several frames. Unreal exposes this directly: s.LevelStreamingActorsUpdateTimeLimit caps the time spent in AddToWorld per frame, and component registration is sliced by a granularity count (s.LevelStreamingComponentsRegistrationGranularity); set the limit too high and "you will see a lot of hitches if your levels are content heavy"[9]. A second, distinct main-thread spike is the garbage collector running after a cell streams out; Unreal can force a full collection after a level streams out or run it incrementally while levels wait to be purged, and the incremental setting avoids the spike[9]. The disk is a cause of stalls on a slow HDD or when bandwidth-starved; on modern hardware, integration is the more common one, and the two need different fixes.

Worked example

The widget below models a frame whose other work costs 9 ms, and a cell that brings 14 ms of activation. All at once, that frame takes 9 + 14 = 23 ms: 6.3 ms over the 16.7 ms line, one visible hitch per cell. With a 2 ms budget the same work takes 14 / 2 = 7 frames of 11 ms each, and the cell goes live 7 frames, about 117 ms, later. The frame has 16.7 − 9 = 7.7 ms of slack, so the budget has to stay under that: at 8 ms, the sliced frames take 17 ms and hitch anyway.

A cell finishes loading every second or so and brings a chunk of main-thread integration work. Integrate it all at once and the frame bar spikes over the 16.7 ms line (a hitch); time-slice it and the frame stays smooth while the cell takes a few extra frames to go live:
Budget 2.0 ms/frame
time-slice integration
load a cell now
Watch the frame bars scroll, then untick time-slice integration, and drag the budget slider with it ticked. The model: the rest of the frame costs 9 ms, each cell brings 14 ms of activation work, and a cell finishes loading about every 1.1 s. The bytes are already in memory in both modes, so the disk plays no part. All-at-once: the cell's activation lands on one frame and the bar punches through 16.7 ms (a hitch). Time-sliced: the same work is metered out under a per-frame budget, so frames stay smooth and the cell goes live a few frames later. A smaller budget is smoother but slower to finish a cell, and the budget has to fit the frame's slack: above about 7.7 ms, even time-sliced frames cross the line.
The integration budget: drain activation work to a per-frame time limit
C++
Rust

// Worker thread (NOT the frame thread): read the cell, then queue it for activation.
void onCellLoaded(Cell& cell, Bytes raw) {
    cell.deserialize(raw);                  // parse off-thread; still not "live"
    std::lock_guard lock(handoffMutex);     // workers and the frame thread share this list
    loadedCells.push_back(&cell);           // hand to the main thread
}

// Main thread, once per frame: spend only the budget, finish the rest next frame.
void integrateStreamedCells(std::chrono::microseconds budget) {
    {
        std::lock_guard lock(handoffMutex); // take what the workers finished; lock held briefly
        pendingIntegration.insert(pendingIntegration.end(), loadedCells.begin(), loadedCells.end());
        loadedCells.clear();
    }
    auto sliceStart = std::chrono::steady_clock::now();
    while (!pendingIntegration.empty()) {
        if (std::chrono::steady_clock::now() - sliceStart > budget) break;  // time-slice (cf. UE's update time limit)
        Cell* cell = pendingIntegration.front();
        if (cell->integrateNextChunk())     // register a few components / upload one batch
            pendingIntegration.pop_front(); // the cell is fully live
    }
}

What's intentionally missing

This skips the production concerns the streamer also needs: a pin list so a cell in use this frame can't be evicted mid-frame; GPU upload fences (issuing the copy is not the same as the copy being done); dependency ordering so a cell's textures are resident before a mesh references them; double-buffered staging; bandwidth-aware throttling; and decompression-worker backpressure. The budget check also runs between chunks, so one long chunk still overshoots; engines keep chunks small for that reason (Unreal's registration granularity above). The File Streaming tutorial owns the residency-pool and eviction internals this builds on.
