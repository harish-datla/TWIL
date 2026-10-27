# Searching millions of vectors

Once every document chunk has an embedding, answering a query means finding the stored vectors closest to the query's vector. The simple way is to compare the query with every stored vector and keep the best.

Worked example

10 million chunks with 768-number embeddings. One query costs 10,000,000 × 768 ≈ 7.7 billion multiply-adds. Stored as 4-byte floats, the vectors alone take 10,000,000 × 768 × 4 bytes ≈ 31 GB.

That brute-force search is exact, and it's often the right choice. The Faiss guidelines say that for only a few thousand searches, building an index doesn't pay for itself and direct comparison is the most efficient option. Faiss calls this a flat index and uses it as the exact baseline every approximate index is measured against.[8] At higher query rates an approximate index gives up a little accuracy for a large speed-up. When memory is the limit, product quantization compresses each vector to a short code of M bytes, typically 64 or fewer, at a further cost in accuracy.[8]

The accuracy measure for an approximate index is recall: the fraction of queries for which it returns the true nearest neighbour, the one brute force would find. "Recall@1" means only the single top result is checked.

HNSW (hierarchical navigable small world) is the index Faiss recommends when there's enough memory.[8] It's a graph: each stored vector is a node linked to some of its near neighbours. A search walks the graph, always stepping to whichever linked node is closer to the query, until no neighbour is closer. On a big dataset that walk would take many short steps, so HNSW adds sparser layers on top that work like express lanes. A random subset of nodes also lives in layer 1, a subset of those in layer 2, and so on. A query starts at the top, makes a few long jumps, drops a layer, and repeats until it reaches the bottom.

Each node's top layer is drawn at random as ⌊−ln(U)/ln(M)⌋, where U is uniform between 0 and 1 and M is the number of links per node.[9] That gives each node a 1 in M chance of reaching layer 1, a 1 in M² chance of reaching layer 2, and so on.

Worked example

M = 8, so ln 8 ≈ 2.08. A draw of U = 0.5 gives −ln 0.5 / 2.08 = 0.33, which rounds down to layer 0. U = 0.05 gives 1.44: layer 1. U = 0.01 gives 2.21: layer 2.

Out of 400 points, about 400 / 8 = 50 reach layer 1 and 400 / 64 ≈ 6 reach layer 2. The widget's random draw gives 48 and 10.

Two settings control the trade-offs. M is the number of links per node: up to M in each upper layer and, in hnswlib, up to 2M in the bottom layer.[9] Higher M builds a better-connected graph and costs roughly M × 8 to 10 bytes of memory per stored vector, with 12 to 48 covering most workloads. ef is the beam width at the bottom layer: instead of following one best candidate, the search keeps the ef closest nodes it has seen and keeps expanding them. Higher ef means more distance computations and higher recall.[10]
Query a small HNSW graph and count the distances it computes
Links per node M
Beam width ef
4
Query x
0.62
Query y
0.37
Reset
400 points, M = 8, ef = 4. Layers hold 400 / 48 / 10 / 1 points from the bottom up. The search descended 3 upper layers and evaluated 61 distances (brute force: 400). Returned point 8; true nearest neighbour 8: found.

Move the query (the pink dot) with the sliders and change ef and M. The yellow path joins the points where the search dropped down a layer, and the green dot is the point it returned. The index is built in the page with the level rule above, ef_construction = 40 (the beam width used while building) and up to 2M links in the bottom layer; it links each new point to the nearest candidates found, without hnswlib's neighbour-diversity heuristic. The right panel shows layer sizes falling by about a factor of M per layer, which is what the level formula produces. The bottom-layer beam search isn't drawn. In two dimensions the graph is easy to navigate: over a grid of query positions, M = 8 and M = 16 find the true nearest neighbour almost every time, even at ef = 1, while evaluating on average roughly 45 to 160 distances out of 400. M = 4 misses about one query in seven at ef = 1. The next widget shows what changes when the vectors have sixteen coordinates.
Trade distance evaluations for recall in sixteen dimensions
Links per node M
Reset
500 random 16-dimensional points, M = 8, 60 queries. ef 1: recall@1 0.55 at 57 distances; ef 2: recall@1 0.70 at 71 distances; ef 4: recall@1 0.78 at 92 distances; ef 8: recall@1 0.93 at 123 distances; ef 16: recall@1 1.00 at 177 distances; ef 32: recall@1 1.00 at 250 distances; ef 64: recall@1 1.00 at 344 distances; ef 128: recall@1 1.00 at 436 distances. Brute force: recall 1 at 500.

Pick M and read the curve: each dot is one ef setting, from 1 up to 128, placed by its recall@1 and the average number of distances it computed per query. Sixty queries against 500 random 16-dimensional points, recall measured against brute force. With M = 8, ef = 1 finds the true neighbour 55% of the time at 57 distances; ef = 8 reaches 93% at 123; ef = 16 reaches 100% at 177, about a third of brute force's 500. M = 4 needs a wider beam for the same recall, and M = 16 a narrower one at more distances per step, which is the memory-versus-speed trade the hnswlib notes describe. ann-benchmarks plots the same trade for real implementations, with queries per second in place of distance evaluations.[11]
