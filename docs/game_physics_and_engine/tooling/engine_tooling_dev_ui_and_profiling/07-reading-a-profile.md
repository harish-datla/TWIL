# Reading a profile

The profile says update takes 6 ms, and a teammate starts rewriting it. That number doesn't say the code in update is slow. A function's total (inclusive) time counts everything it calls; its self time (exclusive) counts only its own instructions. Tracy's statistics view offers both, as "With children" and "Self only"[9]. A function that only calls other functions can have a large total and almost no self time. Optimize where self time piles up.

The same profile data can be drawn two ways. A flame chart puts time on the x-axis, so it shows what ran in what order and lets you find the one slow frame. Brendan Gregg's flame graph merges identical call stacks: its x-axis is "sorted alphabetically (it is not the passage of time)", a box's width is how often that function was on the sampled stacks, and the top edge shows what was running[16]. Merging is what makes it useful: a thousand short calls from the same caller become one wide box.

Merging only joins identical stacks, though. A function called from six places still shows up as six separate boxes, each one narrow. To see its whole cost, list functions by self time; most profilers have this "by function" (or bottom-up) view.

Worked example

In the frame below, alloc never runs for more than 0.5 ms at a stretch, so no single alloc box in the timeline looks worth chasing. Added up, its seven calls take 2.9 ms of the 15.6 ms frame, 19%, all of it self time, more than any other function. update has the largest total of any system, 6.0 ms, and 0.0 ms of self time: rewriting update's own code can't save anything.
timeline
merged
by function
highlight
alloctotal 2.9 ms (19% of the frame)self 2.9 ms (19%) · 7 boxes in this view
You're looking at one recorded frame, three ways. Click a box or pick a function to highlight it everywhere; the readout gives its total and self time. Timeline is the flame chart: alloc shows up as seven slivers. Merged is the flame graph: record's two alloc calls fold into one box, leaving six, and the systems sort A to Z (physics, render, update), so left to right no longer means earlier to later. Widths are each stack's share of the frame, the numbers a sampler's counts converge to. By function sorts by self time, and alloc leads with 2.9 ms. Pick update to see 6.0 ms of total time and none of its own.
