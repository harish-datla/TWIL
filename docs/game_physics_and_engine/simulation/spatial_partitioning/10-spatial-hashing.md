# Spatial hashing

An open-world map is too big for a dense grid array, but most of its cells are empty at any moment. A spatial hash is a grid stored as a hash table instead of a dense 2D/3D array. Insert: compute the cell coordinates from the object's position, hash them to a table index, append the object to that bucket. Query: compute which cells overlap the query region, look up each bucket, test the contents.

Cell size selection matters. If cells are too large, each cell contains too many objects and the query degenerates to brute force within the cell. If cells are too small, objects span multiple cells (the large-object problem again). A common heuristic sets the cell size near the typical object diameter (twice the average radius): an object that small can overlap at most two cells per axis, so at most four in 2D and eight in 3D, and a neighbor search around a point needs only the surrounding 3×3 block.

Worked example

With 1 m cells, a 1 m-wide crate centered at (12.3, 7.8) spans x = 11.8 to 12.8 and y = 7.3 to 8.3, so it lands in cells x = 11 and 12 by y = 7 and 8: four cells, the most a 1 m object can touch in 2D. A 10 m truck in the same grid touches up to 11 × 11 = 121 cells.

Multi-level hashing uses several hash tables at different cell sizes (e.g., 1x, 4x, 16x). Small objects go in the fine grid, large objects in the coarse grid. A query checks all levels. Each object then lands in a level where it spans only a few cells, which keeps mixed-size scenes out of the large-object problem at the cost of one lookup per level.
Spatial hash insert + query
C++
Rust

#include <cmath>
#include <cstdint>
#include <vector>

struct Entity { float posX, posY; };
struct AABB   { float minX, minY, maxX, maxY; };

struct SpatialHash {
    float cellSize;
    int tableSize;
    std::vector<std::vector<Entity*>> buckets;

    SpatialHash(float cellSize, int tableSize)
        : cellSize(cellSize), tableSize(tableSize),
          buckets(tableSize) {}

    // Hash cell coordinates to a bucket index.
    // The primes are Teschner et al.'s; the unsigned cast matters.
    // On int, cellX * 73856093 overflows at cellX = 30, and signed overflow is
    // undefined behavior, so the compiler is free to assume it never happens.
    // Unsigned arithmetic is defined to wrap, which is what we actually want.
    uint32_t hash(int cellX, int cellY) const {
        uint32_t hx = static_cast<uint32_t>(cellX) * 73856093u;
        uint32_t hy = static_cast<uint32_t>(cellY) * 19349663u;
        return hx ^ hy;   // caller reduces modulo tableSize
    }

    int toCell(float coord) const {
        return static_cast<int>(std::floor(coord / cellSize));
    }

    void insert(Entity* entity) {
        int cx = toCell(entity->posX);
        int cy = toCell(entity->posY);
        int idx = static_cast<int>(hash(cx, cy) % static_cast<uint32_t>(tableSize));
        buckets[idx].push_back(entity);
    }

    // Range query: find all entities within queryBox.
    void query(const AABB& queryBox,
               std::vector<Entity*>& results) const
    {
        int minCX = toCell(queryBox.minX);
        int maxCX = toCell(queryBox.maxX);
        int minCY = toCell(queryBox.minY);
        int maxCY = toCell(queryBox.maxY);

        for (int cx = minCX; cx <= maxCX; ++cx) {
            for (int cy = minCY; cy <= maxCY; ++cy) {
                int idx = static_cast<int>(hash(cx, cy)
                          % static_cast<uint32_t>(tableSize));
                for (Entity* entity : buckets[idx]) {
                    // Two cells of this query can hash to the same bucket.
                    // Report an entity only from its own cell, or a shared
                    // bucket hands back the same entity twice.
                    if (toCell(entity->posX) != cx || toCell(entity->posY) != cy)
                        continue;
                    // Fine check: is this entity actually in the query?
                    if (entity->posX >= queryBox.minX
                        && entity->posX <= queryBox.maxX
                        && entity->posY >= queryBox.minY
                        && entity->posY <= queryBox.maxY)
                    {
                        results.push_back(entity);
                    }
                }
            }
        }
    }

    void clear() {
        for (auto& bucket : buckets) bucket.clear();
    }
};

What's intentionally missing

The hash is the teaching version. Production versions add:

    A cheaper clear. clear() walks every bucket every frame, which costs O(tableSize) whether or not anything was in them. The standard fix is a per-bucket frame stamp: a bucket whose stamp is stale is treated as empty and overwritten on first touch, so clearing costs nothing.
    Deduplication for objects with extent. These panes store points, so each entity lives in exactly one cell, and the own-cell check in the query is enough to stop a bucket shared by two query cells from reporting it twice. Store objects with extent and one entity occupies several cells, so a range query spanning them meets it several times. Either stamp each entity per query, or accept duplicates and let the narrow phase discard them.
    Bucket layout. A vector per bucket is a separate heap allocation per bucket, which throws away the locality the structure was chosen for. A common production layout is a counting pass followed by a single packed array with per-bucket offsets, the same layout a counting sort produces.
    A cell size that is not a guess. Both panes take cellSize from the caller. Too small and one object spans many cells; too large and every bucket is a linear scan. The usual starting point is roughly the average object diameter, tuned against the actual distribution.

