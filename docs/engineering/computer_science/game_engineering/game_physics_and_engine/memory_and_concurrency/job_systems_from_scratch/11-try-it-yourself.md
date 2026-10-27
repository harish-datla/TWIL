# Try It Yourself


Before you split a real frame into jobs, it pays to see where adding workers stops helping. The playground below runs a JavaScript model of the scheduler above: a greedy scheduler that hands each ready task to the first free worker and advances simulated time, so the timeline shows where parallelism runs out. It models the dependency tracking, not the deques or the stealing. The library is exposed as MPGJobs; you can build task graphs, run them, and see the worker timeline for the result. Hit Run (or Ctrl+Enter / Cmd+Enter) to execute. Output prints below; the worker timeline is drawn on the right.
▸ playground.js · live JavaScript scheduler, in your browser
Reset
▶ Run
// Build a small frame-shaped graph and run it on 4 workers.
const sched = MPGJobs.create({ workers: 4 });

// "work" simulates a job by burning some microseconds.
const work = (ms) => () => MPGJobs.busy(ms);

// Root: gather input.
const input = sched.submit(work(0.4), 'input');

// Three things branch off input.
const ai       = sched.submit(work(1.8), 'AI',       [input]);
const gameplay = sched.submit(work(1.2), 'gameplay', [input]);
const physics  = sched.submit(work(3.0), 'physics',  [input]);

// Render path joins on all three.
const culling  = sched.submit(work(2.5), 'culling',  [physics, gameplay]);
const animation = sched.submit(work(2.0), 'animation', [gameplay]);
const draw     = sched.submit(work(2.4), 'draw lists', [culling, animation, ai]);

// Audio runs alongside everything.
const audio    = sched.submit(work(1.5), 'audio',    [gameplay]);

const present  = sched.submit(work(0.3), 'present',  [draw, audio]);

const frameMs = await sched.run();   // simulated frame time

console.log(`frame completed in ${frameMs.toFixed(2)} ms`);
console.log(`critical path: input → physics → culling → draw → present`);
console.log(`(try changing the worker count or job sizes)`);

Worked example

The starting graph has nine jobs totaling 15.1 ms of work, and its critical path, input → physics → culling → draw lists → present, is 0.4 + 3.0 + 2.5 + 2.4 + 0.3 = 8.6 ms.

Run it as written and it reports 8.60 ms on 4 workers. Change workers: 4 to 2 and it reports 9.80 ms; at 1 worker, 15.10 ms, the sum of the work. At 8 workers it's still 8.6 ms: the critical path is the floor.

Edit anything. Drop the worker count to 1 (a single-threaded version) and the frame stretches out. Bump it to 8 and the frame shrinks until the critical path sets the length. Try a chain of 30 dependent jobs and you'll see that adding workers doesn't help past the critical-path length.

[Back to Job Systems from Scratch](index.md)
