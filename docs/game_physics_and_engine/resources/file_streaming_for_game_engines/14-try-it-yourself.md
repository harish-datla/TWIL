# Try It Yourself


The playground below runs a simplified JavaScript version of the streamer above, exposed as MPGStream: loads go nearest-first within a bandwidth budget and eviction is LRU. You can drive a simulated player around a world, see tiles loaded and evicted, and tune cache size, bandwidth, and prefetch radius. Hit Run (or Ctrl+Enter / Cmd+Enter). Output prints below; the world view animates on the right.
▸ playground.js · live JavaScript streamer, in your browser
Reset
▶ Run
// Build a 24 x 24 world of 256-KB tiles and a streamer with a 64-tile cache.
const stream = MPGStream.create({
  worldSize: 24,
  tileBytes: 256 * 1024,
  cacheTiles: 64,
  bandwidthMBs: 80,        // megabytes per second the device can deliver
  prefetchRadius: 3,       // prefetch a 7 x 7 square around the player
});

// Drive the player along a path. The streamer reacts to position and velocity.
const path = [
  [2, 2], [4, 3], [7, 4], [11, 6], [15, 9],
  [18, 13], [20, 17], [21, 21],
];
for (const [x, y] of path) {
  stream.movePlayer(x, y);
  await stream.tick(0.25);  // simulate 250 ms of game time
}

const stats = stream.stats();
console.log(`tiles loaded:   ${stats.loaded}`);
console.log(`tiles evicted:  ${stats.evicted}`);
console.log(`pop-in events:  ${stats.popIn}`);
console.log(`peak resident:  ${stats.peakResident} tiles`);
console.log(`(try lowering bandwidthMBs and watch pop-in events spike)`);

"Pop-in events" counts tiles next to the player that aren't resident when the player arrives. Drop bandwidthMBs to 10 and pop-in goes from 0 to about 20: the streamer can't keep up with the player's traversal. Put the bandwidth back and set prefetchRadius to 6 instead: the 169-tile prefetch square no longer fits the 64-tile cache, so tiles loaded jumps from 240 to about 600 and most of them are evicted again before the player gets near them. Neither setting is wrong on its own; the right radius depends on the cache size and bandwidth it has to work with.

[Back to File Streaming for Game Engines](index.md)
