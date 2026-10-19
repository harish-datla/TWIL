# Quadtrees

An RTS map has dense clusters of units around bases and wide stretches of empty ground, and a uniform grid spends as much on the empty ground as on the bases. A quadtree recursively subdivides 2D space into four quadrants. Each internal node has exactly four children (NW, NE, SW, SE). Objects are inserted into the deepest node that fully contains them. When a leaf exceeds a capacity threshold, it subdivides.

Two flavors exist. A point quadtree stores point data; each split plane passes through a stored point. A region quadtree splits space uniformly at the midpoint of each axis, regardless of where the points are. Region quadtrees are the usual choice in engines: the split positions are implicit (computable from the node's bounds), so the tree stores no split-plane data.

Depth limits prevent degenerate input (many objects at the same position) from producing infinite recursion. A typical max depth of 8 gives 2&sup8; = 256 leaf cells per axis, fine resolution for most 2D scenes. The per-leaf capacity (max objects before subdivision) is usually between 4 and 16.

Worked example

On a 1,024 m map with a depth limit of 8, the smallest leaves are 1,024 ÷ 256 = 4 m across. With a capacity of 4, the fifth unit to walk into a leaf splits it into four children half as wide and pushes the five units down into them. An empty corner of the map stays one big leaf, however many units crowd the bases.

In the widget below, each click places an entity, and a leaf subdivides as soon as it overflows.
Live · Quadtree builder
+20 random
clear
max depth 5
max per leaf 4
entities
0
tree nodes
1
max depth reached
0
Entities as purple dots and the quadtree's cells drawn over them, with readouts for entities, tree nodes and the deepest level reached. Click anywhere on the canvas to place an entity, or press +20 random. When a leaf node exceeds the "max per leaf" threshold, it subdivides into four children. The depth slider caps how deep the tree can grow. Cluster entities in one corner to see deep subdivision; spread them evenly to see shallow, balanced splits.
Grid vs quadtree: range query comparison

The practical question is how much each structure touches to answer a range query. A grid visits every cell the query rectangle overlaps, whether those cells hold objects or not. A region quadtree never subdivides empty space, so an empty area is a few large leaves and a query there touches a handful of nodes. Over a dense cluster the balance flips: the quadtree walks down through internal nodes to reach small leaves and can visit more nodes than the grid has cells under the query. For roughly uniform distributions the grid usually wins outright, since it has no traversal at all.
Live · Grid vs Quadtree range query
grid: cells visited · points tested
15 cells · 0 pts
quadtree: nodes visited · points tested
2 nodes · 0 pts
results found
0
Drag the yellow query rectangle, or tap where it should go. The 80 points sit in two clusters in opposite corners. A visited node is one whose box overlaps the query, internal nodes included, which is the quadtree's counterpart of a grid cell under the query. The rectangle starts in an empty corner: the quadtree answers from two nodes (the root and the empty top-right quadrant) while the grid still opens all 15 cells beneath it. Drag it onto a cluster and the count flips: the quadtree descends through 30 to 45 nodes against the grid's 12 to 20 cells, and both end up testing about the same points. Between the clusters, near the middle, the two come out roughly even, because the quadtree has already subdivided around each cluster's edge.
Quadtree insert + range query
C++
Rust

#include <cstddef>
#include <memory>
#include <vector>

struct Entity { float posX, posY; };
struct AABB   { float minX, minY, maxX, maxY; };

struct QuadTreeNode {
    AABB bounds;
    int depth;
    int maxDepth;
    std::size_t maxPerLeaf;   // compared against items.size(), so unsigned like it
    std::vector<Entity*> items;
    std::unique_ptr<QuadTreeNode> children[4]; // NW, NE, SW, SE

    bool contains(float px, float py) const {
        return px >= bounds.minX && px < bounds.maxX
            && py >= bounds.minY && py < bounds.maxY;
    }

    // Children must inherit the limits, or the recursion tests garbage.
    std::unique_ptr<QuadTreeNode> makeChild(AABB childBounds) const {
        auto node = std::make_unique<QuadTreeNode>();
        node->bounds     = childBounds;
        node->depth      = depth + 1;
        node->maxDepth   = maxDepth;
        node->maxPerLeaf = maxPerLeaf;
        return node;
    }

    void subdivide() {
        float midX = (bounds.minX + bounds.maxX) * 0.5f;
        float midY = (bounds.minY + bounds.maxY) * 0.5f;
        children[0] = makeChild({bounds.minX, bounds.minY, midX, midY});
        children[1] = makeChild({midX, bounds.minY, bounds.maxX, midY});
        children[2] = makeChild({bounds.minX, midY, midX, bounds.maxY});
        children[3] = makeChild({midX, midY, bounds.maxX, bounds.maxY});

        // Re-insert existing items into children
        for (Entity* entity : items) {
            for (auto& child : children) {
                if (child->contains(entity->posX, entity->posY)) {
                    child->insert(entity);
                    break;
                }
            }
        }
        items.clear();
    }

    void insert(Entity* entity) {
        if (!contains(entity->posX, entity->posY)) return;

        // Leaf: store here if under capacity or at max depth
        if (!children[0] && (items.size() < maxPerLeaf || depth >= maxDepth)) {
            items.push_back(entity);
            return;
        }
        if (!children[0]) subdivide();

        for (auto& child : children) {
            if (child->contains(entity->posX, entity->posY)) {
                child->insert(entity);
                return;
            }
        }
    }

    // Range query: collect all entities whose positions fall inside queryBox.
    void query(const AABB& queryBox, std::vector<Entity*>& results) const {
        // Prune: if this node does not overlap the query, skip entirely
        if (queryBox.maxX < bounds.minX || queryBox.minX > bounds.maxX ||
            queryBox.maxY < bounds.minY || queryBox.minY > bounds.maxY) return;

        for (Entity* entity : items) {
            if (entity->posX >= queryBox.minX && entity->posX <= queryBox.maxX &&
                entity->posY >= queryBox.minY && entity->posY <= queryBox.maxY) {
                results.push_back(entity);
            }
        }

        if (children[0]) {
            for (const auto& child : children) {
                child->query(queryBox, results);
            }
        }
    }
};

What's intentionally missing

The quadtree above is written to be read. A shipping one adds:

    Removal. There is insert and there is query, and no way to take an entity back out. Removal also needs a merge pass that collapses a node into a leaf once its four children hold fewer than maxPerLeaf items between them, or a tree that has ever been busy stays deep forever.
    Objects with extent. Both panes store a point and test containment. An entity with a bounding box can straddle four children, and the tree has to pick a policy: insert into every overlapping child (duplicates, so queries need a dedup pass), insert into the lowest node that fully contains it (cheap, but a small object sitting on the root split lives at the root forever), or use loose bounds (§11).
    Arena allocation. One unique_ptr per node is a pointer chase per level and an allocation per subdivide. Engines pack nodes into a flat array and store child indices, so a node and its children usually land on the same few cache lines.
    Matching ownership across the panes. The C++ tree stores Entity* and the Rust tree stores Entity by value. Both compile and both are defensible; they are not the same design, and the Rust query hands back references into the tree's own copies rather than into your entity storage.

Check Your Understanding Q1 of 2

You are building collision detection for a 2D particle system with 10,000 particles, all roughly the same size. Which spatial structure fits best?
A quadtree capped at depth 10, subdividing where the particles cluster.
A uniform grid with cell size matched to the particle diameter.
A BVH built with the surface area heuristic at load time.
0/2 correct
