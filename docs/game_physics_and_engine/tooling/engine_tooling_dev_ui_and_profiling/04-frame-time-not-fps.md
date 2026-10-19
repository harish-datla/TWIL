# Frame time, not FPS

QA files a bug: "the game runs at 60 fps but feels choppy." Both halves can be true. An FPS number is an average over many frames, and an average can't show one slow frame among forty fast ones. Players see individual frames, so profiling starts by recording how long each one took in milliseconds, its frame time.

Milliseconds are also the unit that costs add up in. FPS is the reciprocal of frame time, so the same change in FPS means a different amount of work depending on where you start.

Worked example

Dropping from 60 to 50 fps means each frame went from 16.7 ms to 20.0 ms: 3.3 ms of new work. Dropping from 30 to 25 fps means 33.3 ms to 40.0 ms: 6.7 ms, twice as much, for the same "lost 5 fps". Run it the other way: a feature that costs 2 ms takes a 60 fps game to 18.7 ms, 53.6 fps, a loss of 6.4 fps. The same 2 ms takes a 30 fps game to 35.3 ms, 28.3 fps, a loss of 1.7 fps. Budget and compare in milliseconds.

To describe thousands of frames in a few numbers without hiding the slow ones, sort them and read the slow end. The 99th-percentile frame time (P99) is the frame time that 99% of frames beat. A percentile counts frames, not time, and the developers of the CapFrameX benchmarking tool give an example: in a 20-second run, four 500 ms stalls are under 1% of the frames, so a P99 can look fine while the screen was frozen for 2 of the 20 seconds[5]. The measure that counts time is the share of wall-clock time spent inside frames slower than a threshold. The same source points out that a "min FPS" averaged over one second can't show micro-stutter at all, and that the "1% low" figure quoted in benchmark videos has no agreed definition, so compare it only between runs of the same tool[5].

Worked example

In the widget's hitch run, 39 frames of 15.4 ms are followed by one 48 ms frame. The average frame is (39 × 15.4 + 48) / 40 = 16.2 ms, which is 61.7 fps, higher than the steady run's 59.9 fps. The hitches are 1 frame in 40, 2.5% of frames, so the 99th percentile lands on one of them: 48.1 ms. Counted by time, they're 48 / 648.6 = 7.4% of the run.

Three runs with nearly the same average. Switch between them and compare the average with the slow-end numbers:
steady
hitches
uneven
average 59.9 fps (16.7 ms)1-second counter 59 fpsP99 frame 17.1 msworst 17.1 mstime in frames over 20 ms 0.0%
You're looking at the last 120 frame times, newest on the right, with the 60 fps and 30 fps lines; the numbers below the chart cover the last 600 frames. Switch between the three runs. Steady holds 16.7 ms: 59.9 fps, P99 17.1 ms, no time over 20 ms. Hitches has the best average, 61.7 fps, and its 1-second counter reads about 60, but its P99 is about 48 ms and 7.4% of the time goes to frames over 20 ms. Uneven alternates 8.4 ms and 25 ms frames: the average and the counter both say about 60, while about 75% of the time is spent in the 25 ms frames. Frames arrive in real time, so a 48 ms hitch takes 48 ms to show up.

Take the frame time at one fixed point in the loop, such as the top of each frame, from a monotonic clock (the Game Loop page covers which clock). To measure a build without touching its code on Windows, Intel's PresentMon reads the operating system's presentation events and logs, for every frame, the time since the previous Present() call (MsBetweenPresents) and how long the CPU and GPU were busy on it[6].
