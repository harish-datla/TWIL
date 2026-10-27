# A Working Streamer, in Your Language


Below is a complete streamer in two languages: modern C++ (20) and Rust. The C++ version is about 250 lines and uses no dependencies beyond the standard library. The Rust version is similar. Both implement the same design: a pool of worker threads draining a shared request queue with blocking reads (one read in flight per worker), a priority queue that orders requests by score, an LRU pool that bounds residency, and a callback that runs CPU-side decompression. There is no platform-specific I/O; the goal is to read clearly. In production you'd replace the blocking worker reads with io_uring / IoRing / DirectStorage submissions; the priority queue, dedup and residency pool carry over.
streamer · pick a language ↓
C++
Rust
Copy

// ───────────────────────────────────────────────────────────
// streamer.cpp · priority-scheduled async streamer with LRU
// Build: g++ -std=c++20 -O2 -pthread -c streamer.cpp   (a library TU; link it into your game)
// ───────────────────────────────────────────────────────────
#include <atomic>
#include <condition_variable>
#include <cstddef>
#include <cstdint>
#include <fstream>
#include <functional>
#include <list>
#include <memory>
#include <mutex>
#include <queue>
#include <string>
#include <thread>
#include <unordered_map>
#include <unordered_set>
#include <vector>

namespace mpg {

// A stable, content-agnostic identifier (typically a hash of the asset's
// logical name). Pending reads, residency pool lookups, and dedup all
// key on this ID.
using AssetId = uint64_t;

// Everything the streamer needs to fetch and decompress one asset.
struct PendingRead {
  AssetId   assetId;
  float     priorityScore;        // higher = load sooner; computed by caller
  uint64_t  byteOffsetInBundle;   // where in the bundle to start reading
  uint32_t  compressedByteCount;  // bytes to read from disk
  uint32_t  uncompressedByteCount;// bytes after the decoder runs
};

// Comparator for the std::priority_queue. The std heap is a max-heap by
// default; we order by priorityScore so the highest-priority item pops first.
struct HigherPriorityFirst {
  bool operator()(const PendingRead& left, const PendingRead& right) const {
    return left.priorityScore < right.priorityScore;
  }
};

// The data the streamer hands back to the engine once a read finishes.
struct ResidentAsset {
  std::vector<std::byte> decompressedBytes;
};

// A least-recently-used residency pool. The doubly-linked list is
// ordered front=newest, back=oldest. The hash map indexes into the list
// so lookups are O(1) and the "promote to front on access" is also O(1).
class ResidencyPool {
  size_t capacityBytes;            // budget set by the caller; we evict to stay under it
  size_t residentBytes = 0;        // total size of everything currently in the pool

  // shared_ptr so a caller's handle keeps the bytes alive even if a worker
  // evicts the entry on another thread while the caller is still using it.
  struct CacheEntry { AssetId assetId; std::shared_ptr<const ResidentAsset> asset; };
  std::list<CacheEntry> recencyOrder;   // front = most recently used
  std::unordered_map<AssetId, std::list<CacheEntry>::iterator> indexByAssetId;

public:
  explicit ResidencyPool(size_t capacityInBytes) : capacityBytes(capacityInBytes) {}

  // Look up an asset. If found, also bump it to the front of the LRU list.
  // Returns nullptr if not resident.
  std::shared_ptr<const ResidentAsset> touch(AssetId assetId) {
    auto indexEntry = indexByAssetId.find(assetId);
    if (indexEntry == indexByAssetId.end()) return nullptr;
    // splice() moves the node within the same list in O(1).
    recencyOrder.splice(recencyOrder.begin(), recencyOrder, indexEntry->second);
    return indexEntry->second->asset;
  }

  // Add a freshly-decoded asset to the pool. Evict from the LRU tail until
  // we're back under budget.
  void insert(AssetId assetId, ResidentAsset asset) {
    if (indexByAssetId.count(assetId)) return;   // already resident; nothing to do
    residentBytes += asset.decompressedBytes.size();
    recencyOrder.push_front({assetId, std::make_shared<ResidentAsset>(std::move(asset))});
    indexByAssetId[assetId] = recencyOrder.begin();

    // Evict from the back (oldest) until we're under budget again.
    while (residentBytes > capacityBytes && !recencyOrder.empty()) {
      auto& evictionTarget = recencyOrder.back();
      residentBytes -= evictionTarget.asset->decompressedBytes.size();
      indexByAssetId.erase(evictionTarget.assetId);
      recencyOrder.pop_back();
    }
  }

  bool contains(AssetId assetId) const {
    return indexByAssetId.count(assetId) > 0;
  }
};

class Streamer {
  // ── configuration ──────────────────────────────────────────────────
  std::string           bundlePath;          // path to the asset bundle; each worker opens its own handle
  ResidencyPool         residencyPool;       // the LRU cache of decoded assets
  int                   workerCount;         // how many threads pull from the queue

  // ── synchronization for the pending-request queue ─────────────────
  // The mutex protects pendingQueue + alreadyQueuedIds together.
  std::mutex             queueMutex;
  std::condition_variable requestAvailable;    // signalled when a new read lands
  std::priority_queue<PendingRead,
                      std::vector<PendingRead>,
                      HigherPriorityFirst> pendingQueue;
  std::unordered_set<AssetId> alreadyQueuedIds;  // dedup; clears as workers finish

  // ── worker pool ────────────────────────────────────────────────────
  std::vector<std::thread> workerThreads;
  std::atomic<bool>    isRunning{true};      // flipped to false in the destructor

  // The decoder callback turns compressed bytes into decompressed bytes.
  // In production this would dispatch to GDeflate on the GPU or to a
  // hardware block on console.
  std::function<
      std::vector<std::byte>(const std::byte* src, size_t srcByteCount, size_t dstByteCount)
  > decode;

public:
  Streamer(std::string bundleFilePath,
           size_t poolCapacityBytes,
           int concurrentReads,
           auto decoderCallback)
    : bundlePath(std::move(bundleFilePath)),
      residencyPool(poolCapacityBytes),
      workerCount(concurrentReads),
      decode(std::move(decoderCallback)) {
    workerThreads.reserve(workerCount);
    for (int workerIndex = 0; workerIndex < workerCount; workerIndex++)
      workerThreads.emplace_back([this] { workerLoop(); });
  }

  ~Streamer() {
    {
      // Flip the flag under the mutex, same as the naive loader in §4:
      // flipped outside it, a worker can pass the predicate check and go
      // to sleep after the notify below fires, and join() hangs.
      std::lock_guard lock(queueMutex);
      isRunning.store(false);
    }
    requestAvailable.notify_all();        // wake every worker so they can exit
    for (auto& worker : workerThreads) worker.join();
  }

  // Add a tile-load request to the priority queue. Silently dedupes
  // against already-resident assets and already-pending requests.
  void request(PendingRead incomingRequest) {
    std::lock_guard lock(queueMutex);
    if (residencyPool.contains(incomingRequest.assetId)) return;
    if (!alreadyQueuedIds.insert(incomingRequest.assetId).second) return;
    pendingQueue.push(std::move(incomingRequest));
    requestAvailable.notify_one();      // nudge one sleeping worker
  }

  // Engine-facing accessor. Promotes the entry on the LRU list as a side
  // effect. Takes queueMutex because touch() mutates the LRU list while
  // workers insert into the same pool under this lock; unsynchronized,
  // that's a data race. The returned shared_ptr stays valid after the lock
  // drops, even if a worker evicts the entry a moment later.
  std::shared_ptr<const ResidentAsset> access(AssetId assetId) {
    std::lock_guard lock(queueMutex);
    return residencyPool.touch(assetId);
  }

private:
  // One of these runs on each worker thread. Pulls the highest-priority
  // pending read off the queue, executes it, decodes, then stores the
  // result in the residency pool.
  void workerLoop() {
    // Each worker keeps its own ifstream so concurrent seeks don't fight
    // over a single file pointer.
    std::ifstream perWorkerFile(bundlePath, std::ios::binary);

    while (isRunning.load()) {
      // 1. Wait for a request, then pop the highest-priority one.
      PendingRead request;
      {
        std::unique_lock lock(queueMutex);
        requestAvailable.wait(lock, [&] {
          return !pendingQueue.empty() || !isRunning;
        });
        if (!isRunning) return;
        request = pendingQueue.top();
        pendingQueue.pop();
      }

      // 2. Read the compressed bytes from the bundle.
      std::vector<std::byte> compressedBytes(request.compressedByteCount);
      perWorkerFile.seekg(request.byteOffsetInBundle);
      perWorkerFile.read(reinterpret_cast<char*>(compressedBytes.data()),
                          request.compressedByteCount);
      if (!perWorkerFile) {
        // Short read or I/O error (truncated bundle, bad offset). Clear the
        // stream's sticky fail state so this worker's next read can succeed,
        // and drop the dedup mark so a later request for the asset can retry.
        perWorkerFile.clear();
        std::lock_guard lock(queueMutex);
        alreadyQueuedIds.erase(request.assetId);
        continue;
      }

      // 3. Decode on this worker. In production this would dispatch to a
      // GPU compute queue (GDeflate / Zstd) or a hardware block (Kraken).
      auto decompressedBytes = decode(
          compressedBytes.data(),
          request.compressedByteCount,
          request.uncompressedByteCount);

      // 4. Publish the result. Clear the dedup bit so a fresh request for
      // the same asset (e.g. after it gets evicted) can be enqueued again.
      {
        std::lock_guard lock(queueMutex);
        residencyPool.insert(request.assetId,
                              ResidentAsset{std::move(decompressedBytes)});
        alreadyQueuedIds.erase(request.assetId);
      }
    }
  }
};

} // namespace mpg

// ── usage ─────────────────────────────────────────────────────────────
// // A no-op decoder: pretend the on-disk bytes are already uncompressed.
// auto identityDecoder = [](const std::byte* src, size_t srcByteCount, size_t /*dstByteCount*/) {
//     return std::vector<std::byte>(src, src + srcByteCount);
// };
// mpg::Streamer streamer(
//     "assets.pak",
//     /*poolCapacityBytes=*/ 1ull << 30,    // 1 GiB residency budget
//     /*concurrentReads=*/ 16,
//     identityDecoder);
//
// streamer.request({.assetId=tileId, .priorityScore=score, .byteOffsetInBundle=off, ...});
// if (auto asset = streamer.access(tileId)) bindTexture(*asset);

What's intentionally missing

This implementation is meant to read clearly, not be the fastest possible.

    Each worker does blocking reads (ifstream::read / File::read_exact), so queue depth equals the worker count. A real implementation would use io_uring / IoRing / DirectStorage to keep the device queue full from one thread.
    The decompression callback is CPU-side. The PC fast path is a GPU compute shader; the console fast path is fixed-function silicon.
    The residency pool is pure LRU. Production pools have pin lists, tiers, and heat tracking (see §8).
    There's no batching of nearby reads. A real streamer coalesces adjacent requests to amortize per-IO overhead.
    There's no eviction notification. A C++ caller holding a shared_ptr keeps the bytes alive after eviction but isn't told the asset left the pool, and those bytes no longer count against the budget. Production pools pair handles with residency callbacks and count outstanding references toward the budget.
    Failed reads are dropped silently. A real streamer reports the error to whoever requested the asset.


[Back to File Streaming for Game Engines](index.md)
