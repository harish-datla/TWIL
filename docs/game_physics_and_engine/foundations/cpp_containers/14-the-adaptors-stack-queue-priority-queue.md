# The Adaptors: stack, queue, priority_queue


A sound system keeps a list of voices that want to play and, when it runs out of hardware channels, always drops the least important one. That access pattern, "give me the top item", doesn't need a general container: it needs a narrow one that does that job well. The standard library has three such adaptors, thin wrappers that restrict an underlying container to one access pattern:

    std::stack<T>. LIFO. Defaults to std::deque<T> underneath, but any container with back(), push_back(), and pop_back() works. Pass std::vector<T> as the second template argument and you get a vector-backed stack with no deque overhead.
    std::queue<T>. FIFO. Defaults to std::deque<T> underneath. A vector won't work (it has no pop_front, so queue::pop won't compile); a std::list works but is slower than the deque default.
    std::priority_queue<T>. The largest element (by the comparator, std::less by default) is always on top. Defaults to a std::vector<T> arranged as a binary heap by the standard heap algorithms[27].

The binary-heap layout is worth knowing because priority queues are hot in game code: A* pathfinding, sound voice priority, timed-event scheduling, most "do the most important thing next" systems. The heap is a vector where, for the element at index i, its children are at 2i+1 and 2i+2, and the parent is at (i-1)/2. push appends and sifts up (swaps the new element with its parent while it's bigger); pop takes the root, moves the last element to the root, and sifts down (swaps it with its bigger child while that child is bigger).

Worked example

The element at index 3 has children at 2 × 3 + 1 = 7 and 2 × 3 + 2 = 8, and its parent at (3 − 1) / 2 = 1 (integer division).

A heap of 1,000 elements is 10 levels deep, 9 steps from root to bottom (29 = 512 ≤ 1,000 < 1,024). A push compares at most once per level it climbs: 9 comparisons at most. A pop compares with up to two children per level on the way down: 18 at most.
Live · binary heap
value 42
push ▸
pop ▸
seed 10 random ▸
Reset ⟲
size
8
top
91
last op compares
3
The tree and the flat array below it are the same heap; the readouts give its size, its top and the comparisons the last operation needed. Push a value with the slider, or seed 10 random ones, then pop and watch the last element sift down from the root. A binary max-heap stored in a vector. Push and pop are O(log n) (one sift-up or sift-down through the tree). Building a heap from a vector with std::make_heap is O(n), not O(n log n): it sifts down from the bottom up, and most nodes sit near the bottom where a sift-down is short[27]. "Last op compares" counts element comparisons in the most recent push or pop.

The standard library's priority queue is a max-heap by default. For Dijkstra and A*, where you want the smallest distance first, pass std::greater<T> as the comparator:
astar.cpp · min-heap for shortest-path search

struct PathNode {
  NodeId nodeId;
  float  estimatedTotalCost;        // f = g + h: A* total estimate through this node
};
struct SmallerFirst {
  bool operator()(const PathNode& lhs, const PathNode& rhs) const {
    // priority_queue puts the "largest" element on top; comparing with >
    // makes the smallest cost the largest, so it ends up on top.
    return lhs.estimatedTotalCost > rhs.estimatedTotalCost;
  }
};

std::priority_queue<PathNode, std::vector<PathNode>, SmallerFirst> openSet;

// Sketch of the queue's role only: the closed set, the g-cost table and
// its updates, and skipping stale entries are left out.
openSet.push({startNodeId, heuristic(startNodeId, goal)});
while (!openSet.empty()) {
  PathNode currentNode = openSet.top();   // O(1): best frontier node
  openSet.pop();                            // O(log n): sift the last element down
  if (currentNode.nodeId == goal) break;
  for (NodeId neighbor : neighborsOf(currentNode.nodeId))
    openSet.push({neighbor, gCostSoFar(neighbor) + heuristic(neighbor, goal)});
}

priority_queue's missing API

std::priority_queue does not let you change the priority of an element in place; it has no "decrease-key" operation[27]. A Dijkstra or A* implementation pushes a fresh entry with the lower cost and skips stale entries when they reach the top. That lazy approach is usually fast enough in practice. When stale entries bloat the heap (dense graphs where nodes improve many times), an indexed binary heap that tracks each node's position supports decrease-key directly.

[Back to C++ Containers](index.md)
