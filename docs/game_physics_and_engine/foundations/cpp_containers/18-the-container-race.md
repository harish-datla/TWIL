# The Container Race


Every section so far has argued from layout. This widget measures it. It times three workloads (push N elements, iterate them, look up random ones) across five containers. Each container is modeled in JavaScript with typed arrays laid out the way the C++ container lays out memory: the vector is one growing block, the deque is 128-element chunks (libstdc++'s 512 bytes of int), and the list, map and unordered_map are nodes placed at shuffled positions in a pool, the way heap allocations scatter. Every number is a real measurement in your browser, with no calibration factors. Absolute times aren't native C++ times: JavaScript adds bounds checks, and the pool allocation charges nothing for malloc, which understates push costs for the node-based containers. Pick a workload and a size:
Live · five-container race
push N
iterate N
random access
N elements 100,000
RACE ▶
Reset ⟲
vector ms
···
deque ms
···
list ms
···
map ms
···
unordered ms
···
Each bar is one container's measured time for the chosen workload, on a log scale, and each readout also shows the multiple of the vector's time. Pick a workload, set N and press RACE. Iteration shows the layout effect most cleanly: vector and deque stream through contiguous memory, while list, map and unordered_map chase pointers to shuffled nodes. How far apart the bars land depends heavily on your CPU's cache sizes. On a desktop CPU, expect the node-based containers to iterate roughly 10-40× slower than the vector at N = 100,000, where their pools still fit in L2 or L3, and roughly 80-200× slower at 1,000,000, once they spill to DRAM. Random access widens the spread: vector and deque index directly, unordered_map hashes to one bucket (tens of times the vector), map walks O(log n) scattered nodes (hundreds of times), and list has to walk from begin() for every query; ≈ marks a list time extrapolated from a subset of the queries. Push N is the one workload where the list keeps pace with the vector, because this model's pool hands out nodes for free; a real std::list pays a malloc per node. Large N takes a second or two to run.

[Back to C++ Containers](index.md)
