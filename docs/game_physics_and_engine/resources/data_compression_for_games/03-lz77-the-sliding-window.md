# LZ77: The Sliding Window

A level file repeats itself constantly: component names, vertex layouts, and material blocks occur on hundreds of props. A byte-frequency coder cannot see that a 40-byte block appeared 300 bytes earlier. LZ77 can. It emits literals (raw bytes) and back-references: `(distance, length)` means “copy `length` bytes from `distance` bytes back.” The copy comes from a sliding window of recently decoded output.

## Worked example

To encode `blah blah blah`, the first five characters (`b`, `l`, `a`, `h`, and the space) are literals. At position 5, the encoder finds `blah blah` five characters back and emits `(distance 5, length 9)`. Because the length is greater than the distance, the decoder copies bytes it has only just written. In a teaching cost model—9 bits per literal and 18 bits per match—the result costs `5 × 9 + 18 = 63` bits, about 8 bytes for 14 characters.

- A back-reference points into already-decoded output, not a future position or external dictionary.
- Overlapping matches are legal and encode runs efficiently.
- zstd, LZ4, DEFLATE, Brotli, and Oodle Kraken are LZ77-based families, but their streams are not interchangeable.
- A larger window can improve ratio at the cost of memory and search work.

## Teaching encoder

```cpp
struct Token { bool isMatch; uint8_t literal; int distance; int length; };

std::vector<Token> lz77Encode(const std::vector<uint8_t>& input,
                              int windowSize, int minMatch) {
    std::vector<Token> tokens;
    int position = 0;
    while (position < (int)input.size()) {
        int bestLength = 0, bestDistance = 0;
        int windowStart = std::max(0, position - windowSize);
        for (int candidate = windowStart; candidate < position; ++candidate) {
            int length = 0;
            while (position + length < (int)input.size() &&
                   input[candidate + length] == input[position + length]) ++length;
            if (length > bestLength) {
                bestLength = length;
                bestDistance = position - candidate;
            }
        }
        if (bestLength >= minMatch) {
            tokens.push_back({true, 0, bestDistance, bestLength});
            position += bestLength;
        } else {
            tokens.push_back({false, input[position], 0, 0});
            position += 1;
        }
    }
    return tokens;
}
```

This intentionally scans naively at `O(n · window · match length)`. Production codecs use hash chains, trees, suffix structures, lazy matching, real bitstreams, match-length limits, and bounds-checked decoders. The example exists only to make the mechanism visible.
