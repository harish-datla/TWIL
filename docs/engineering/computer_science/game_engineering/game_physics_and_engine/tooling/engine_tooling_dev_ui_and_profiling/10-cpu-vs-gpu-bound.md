# CPU- vs GPU-bound

You spend a week speeding up game logic and the frame rate doesn't move, because the GPU was the slow side all along. The CPU and GPU work in parallel, with the CPU usually recording a frame or more ahead of the GPU, so the slower of the two sets the frame time. Bound-ness is about which timeline is the critical path. If the GPU's queue has visible gaps, idle waiting on CPU submission, you're CPU-bound; if the CPU blocks at present/fence waiting for the GPU, you're GPU-bound[24]. A quick test: halve the render resolution. If the frame rate jumps, you were GPU-bound on fragment/fill work (fewer pixels to shade); if it barely moves, you're CPU-bound or bound on GPU geometry/draw submission, which fewer pixels don't change.
Vsync masquerades as GPU-bound

With vsync on, both the CPU and GPU can finish early and then wait for the display refresh, so both timelines show idle time that is waiting for scanout, not for each other, and the app can look CPU- or GPU-bound when it's actually neither. A frame that misses a refresh waits for the next one, so with 60 Hz FIFO vsync and double buffering, 17 ms of work shows up as a 33.3 ms frame. Diagnose with vsync off, so a frame cap isn't mistaken for a bottleneck.

Worked example

Take the widget's starting values: 8 ms of CPU work and 12 ms of GPU work. Without vsync the frame takes the longer of the two, 12 ms (83 fps), and the CPU waits 4 ms at present every frame: GPU-bound, so faster game logic won't help. Turn vsync on at 60 Hz and the frame becomes 16.7 ms (60 fps), with both lanes idle until the refresh: neither is the bottleneck. Drag the GPU to 17 ms with vsync still on and it misses the refresh by 0.3 ms, so the frame waits for the next one: 33.3 ms, 30 fps.

Drag the CPU and GPU work; see which is the critical path and who waits. Toggle vsync and both lanes idle to the next refresh:
CPU work 8ms
GPU work 12ms
vsync
Drag the CPU and GPU sliders and watch which lane goes gray, then tick vsync; the readout under the lanes gives the frame time and the diagnosis. The longer lane is the critical path. GPU > CPU: the CPU finishes and stalls at present (GPU-bound). CPU > GPU: the GPU starves waiting for the next submit (CPU-bound). Turn on vsync and both lanes idle to the next refresh. That's the trap: under 16.7 ms of work the frame looks bound but is only frame-capped, and just over it the frame snaps to 33.3 ms. Disable vsync to tell. (This models FIFO vsync with double buffering; mailbox and triple buffering behave differently.)

For a build you can't instrument, PresentMon gives the same diagnosis from outside on Windows: alongside each frame's time it reports how long the CPU and the GPU were busy on that frame (MsCPUBusy and MsGPUBusy) and how long each waited[6]. A GPU that stays busy for nearly the whole frame time points to GPU-bound; long GPU waits point to the CPU.
