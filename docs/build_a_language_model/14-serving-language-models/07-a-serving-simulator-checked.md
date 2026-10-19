# A serving simulator, checked

The program below, in C++ and Rust, runs the mechanisms above at toy scale and asserts their properties. It has four parts.

    A block pool of 16 blocks of 4 tokens. It builds block tables for prompts, shares full blocks whose chained hash is already cached, counts references, frees blocks to the tail of a least-recently-used queue in reverse order, and evicts cached content from its head, as vLLM's design document describes.[13] Two requests that share a 12-token system prompt must share three blocks, and the pool must be empty again at the end.
    Batching: a 40-request stream runs through 8 slots under both policies, and continuous batching must finish no later and with lower mean latency.
    Admission: 64 sequences are fitted into a token budget, once by reservation and once by blocks.
    Speculation: 200,000 simulated rounds at α = 0.8, γ = 4 must match the expectation formula within 0.02 tokens.

Paged KV pool, batching policies and speculative yield, complete programs
C++20
Rust 2021

// A serving loop in miniature: a paged KV-cache block pool with prefix sharing and LRU eviction, continuous
// versus static batching over one request stream, and the speculative-decoding expectation checked by simulation.
// Build: g++ -std=c++20 -O2 -Wall -Wextra -Wpedantic -Werror serving.cpp && ./a.out
#include <algorithm>
#include <cassert>
#include <cmath>
#include <cstdint>
#include <cstdio>
#include <deque>
#include <map>
#include <vector>

struct Rng {  // xorshift32 with a fixed seed
    uint32_t state = 9;
    double next() { state ^= state << 13; state ^= state >> 17; state ^= state << 5; return state / 4294967296.0; }
    int between(int low, int high) { return low + int(next() * (high - low + 1)); }
};

// ── Paged KV cache: fixed-size blocks, reference counts, content hashes chained through the prefix, LRU free queue.
constexpr int kBlockSize = 4;
struct Block { int referenceCount = 0; uint64_t hash = 0; bool cached = false; };
struct BlockPool {
    std::vector<Block> blocks;
    std::deque<int> freeQueue;                 // head = least recently used
    std::map<uint64_t, int> cachedBlocks;      // full-block hash → block id
    int cacheHits = 0, evictions = 0;
    explicit BlockPool(int count) : blocks(count) { for (int id = 0; id < count; ++id) freeQueue.push_back(id); }
    static uint64_t chainHash(uint64_t parent, const std::vector<int>& tokens, std::size_t begin, std::size_t end) {
        uint64_t hash = parent * 1099511628211ULL + 14695981039346656037ULL;   // parent hash first: a block's identity includes everything before it
        for (std::size_t i = begin; i < end; ++i) hash = (hash ^ uint64_t(tokens[i] + 1)) * 1099511628211ULL;
        return hash;
    }
    // Remove a block from the free queue (it is about to be referenced again).
    void touch(int id) { freeQueue.erase(std::find(freeQueue.begin(), freeQueue.end(), id)); }
    // Pop the least recently used free block; if it still holds cached content, that content is evicted.
    int popFree() {
        assert(!freeQueue.empty());
        const int id = freeQueue.front(); freeQueue.pop_front();
        if (blocks[id].cached) { cachedBlocks.erase(blocks[id].hash); blocks[id].cached = false; ++evictions; }
        return id;
    }
    // Allocate the block table for a prompt: full blocks whose hash is cached are shared, the rest come from the free queue.
    std::vector<int> allocate(const std::vector<int>& tokens) {
        std::vector<int> table; uint64_t parent = 0;
        const std::size_t fullBlocks = tokens.size() / kBlockSize;
        for (std::size_t block = 0; block < fullBlocks; ++block) {
            parent = chainHash(parent, tokens, block * kBlockSize, (block + 1) * kBlockSize);
            if (auto hit = cachedBlocks.find(parent); hit != cachedBlocks.end()) {
                if (blocks[hit->second].referenceCount++ == 0) touch(hit->second);   // in use again: keep it off the free queue
                table.push_back(hit->second); ++cacheHits; continue;
            }
            const int id = popFree(); blocks[id] = {1, parent, true}; cachedBlocks[parent] = id; table.push_back(id);
        }
        if (tokens.size() % kBlockSize) { const int id = popFree(); blocks[id] = {1, 0, false}; table.push_back(id); }   // partial last block: not cacheable yet
        return table;
    }
    // Release a table: blocks with no other users return to the free queue tail, last block first, keeping their cached content until evicted.
    void release(const std::vector<int>& table) {
        for (auto it = table.rbegin(); it != table.rend(); ++it) if (--blocks[*it].referenceCount == 0) freeQueue.push_back(*it);
    }
    int inUse() const { int count = 0; for (const Block& block : blocks) count += block.referenceCount > 0; return count; }
};

// ── Requests and the two batching policies.
struct Request { int id, arrival, prompt, output, remaining, start = -1, finish = -1; };
struct Outcome { int steps; double meanLatency, utilization; };
Outcome simulate(std::vector<Request> queue, int slots, bool continuous) {
    std::vector<Request> active, finished; int step = 0; long generated = 0;
    while (!queue.empty() || !active.empty()) {
        const bool canAdmit = continuous || active.empty();   // static batching waits for the whole batch to drain
        while (canAdmit && int(active.size()) < slots && !queue.empty() && queue.front().arrival <= step) { queue.front().start = step; active.push_back(queue.front()); queue.erase(queue.begin()); }
        for (Request& request : active) { --request.remaining; ++generated; if (request.remaining == 0) request.finish = step + 1; }
        for (auto it = active.begin(); it != active.end();) if (it->remaining == 0) { finished.push_back(*it); it = active.erase(it); } else ++it;
        ++step;
    }
    double latency = 0; for (const Request& request : finished) latency += request.finish - request.arrival;
    return {step, latency / finished.size(), double(generated) / (double(step) * slots)};
}

int main() {
    Rng rng;
    // 1. Block pool: two requests sharing a 12-token system prompt share its three full blocks; frees and evictions balance.
    BlockPool pool(16);
    std::vector<int> systemPrompt(12); for (int& token : systemPrompt) token = rng.between(0, 99);
    std::vector<int> first(systemPrompt), second(systemPrompt);
    for (int i = 0; i < 7; ++i) first.push_back(rng.between(0, 99));     // 19 tokens: 4 full blocks + 1 partial
    for (int i = 0; i < 5; ++i) second.push_back(rng.between(0, 99));    // 17 tokens: 4 full blocks + 1 partial, first 3 shared
    std::vector<int> tableFirst = pool.allocate(first), tableSecond = pool.allocate(second);
    assert(tableFirst.size() == 5 && tableSecond.size() == 5);
    assert(std::equal(tableFirst.begin(), tableFirst.begin() + 3, tableSecond.begin()));   // the shared prefix maps to the same blocks
    assert(pool.cacheHits == 3 && pool.inUse() == 7);                                   // 5 + 5 tables, 3 blocks shared
    pool.release(tableFirst); pool.release(tableSecond);
    assert(pool.inUse() == 0 && pool.freeQueue.size() == 16);
    std::vector<int> again = pool.allocate(first);                                       // everything full is still cached
    assert(pool.cacheHits == 7 && pool.inUse() == 5);
    pool.release(again);
    std::vector<std::vector<int>> churn;                                                // fill the pool with new content to force LRU eviction
    for (int round = 0; round < 4; ++round) { std::vector<int> prompt(16); for (int& token : prompt) token = rng.between(100, 199); churn.push_back(pool.allocate(prompt)); }
    assert(pool.evictions > 0 && pool.inUse() == 16);
    for (const auto& table : churn) pool.release(table);
    assert(pool.inUse() == 0);
    std::printf("block pool: %d cache hits, %d evictions, all blocks returned\n", pool.cacheHits, pool.evictions);

    // 2. Batching: the same 40 requests through 8 slots, admitted only between batches versus whenever a slot frees.
    std::vector<Request> stream; int clock = 0;
    for (int id = 0; id < 40; ++id) { clock += rng.between(0, 6); const int output = rng.between(8, 64); stream.push_back({id, clock, rng.between(16, 128), output, output}); }
    const Outcome fixed = simulate(stream, 8, false), rolling = simulate(stream, 8, true);
    std::printf("static batching: %d steps, mean latency %.1f steps, slot utilization %.2f\n", fixed.steps, fixed.meanLatency, fixed.utilization);
    std::printf("continuous batching: %d steps, mean latency %.1f steps, slot utilization %.2f\n", rolling.steps, rolling.meanLatency, rolling.utilization);
    assert(rolling.steps <= fixed.steps && rolling.meanLatency < fixed.meanLatency);

    // 3. Memory: reserving the maximum length per sequence versus paging, for 64 sequences of 50 to 500 tokens in a 32,768-token budget.
    const int budget = 32768, maxLength = 2048, blockSize = 16;
    int reservedFits = std::min(64, budget / maxLength), pagedFits = 0, blocksUsed = 0;
    for (int i = 0; i < 64; ++i) { const int length = rng.between(50, 500), blocks = (length + blockSize - 1) / blockSize; if ((blocksUsed + blocks) * blockSize > budget) break; blocksUsed += blocks; ++pagedFits; }
    std::printf("memory: %d sequences fit with reservation, %d with %d-token blocks\n", reservedFits, pagedFits, blockSize);
    assert(pagedFits > reservedFits);

    // 4. Speculative decoding: with γ drafts each accepted with probability α, a target pass yields (1 − α^(γ+1)) / (1 − α) tokens on average.
    const double alpha = 0.8; const int gamma = 4, trials = 200000; long tokens = 0;
    for (int trial = 0; trial < trials; ++trial) { int accepted = 0; while (accepted < gamma && rng.next() < alpha) ++accepted; tokens += accepted + 1; }   // +1: the correction or bonus token
    const double measured = double(tokens) / trials, expected = (1 - std::pow(alpha, gamma + 1)) / (1 - alpha);
    std::printf("speculative decoding: %.3f tokens per target pass measured, %.3f expected\n", measured, expected);
    assert(std::fabs(measured - expected) < 0.02);
    return 0;
}

What's intentionally missing

There's no model in the simulator: every step takes one unit of time, prefill is free, and the pool stores token ids rather than keys and values. It also skips preemption and recomputation when the pool runs out, swapping blocks to host memory, copy-on-write for sequences that fork during beam search, vLLM v1's append-only block tables and duplicate-block handling, chunked prefill, priorities and fairness between tenants, and the real speculative rejection sampler that preserves the target's distribution when sampling. The allocation, sharing, eviction and admission rules are the ones the cited engines describe.
