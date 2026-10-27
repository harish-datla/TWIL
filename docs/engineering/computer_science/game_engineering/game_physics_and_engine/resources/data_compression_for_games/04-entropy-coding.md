# Entropy Coding

After LZ77, the token stream is still lopsided: some literals and match lengths occur much more often than others. Entropy coding gives frequent symbols shorter bit patterns, pushing the average toward the entropy floor. Huffman coding builds the best prefix code for a set of frequencies: no code is the beginning of another, so the decoder knows where each symbol ends.

## Building a Huffman tree

Put every symbol in a pile with its count. Remove the two rarest, join them under a node whose count is their sum, and put that node back. Repeat until one tree remains. A symbol’s code is its path from the root: `0` for a left branch and `1` for a right branch. Rare symbols are joined first, so they sit deepest and get longer codes.

## Worked example

For 76 symbols with counts A=35, B=20, C=12, D=6, and E=3, a fixed-width code uses `76 × 3 = 228` bits. Huffman coding can assign A=`0`, B=`10`, C=`111`, D=`1101`, and E=`1100`:

```text
35 × 1 + 20 × 2 + 12 × 3 + 6 × 4 + 3 × 4 = 147 bits
```

That is 1.93 bits per symbol, close to the entropy of 1.92 bits per symbol.

## Beyond Huffman

Huffman uses whole bits per symbol. A symbol with probability 0.9 still costs at least one bit even though its ideal cost is about 0.15 bits. Arithmetic coding, range coding, and modern ANS (asymmetric numeral systems) allow fractional average costs. ANS is table-driven and fast, which is why it is used as an entropy backend in zstd. Entropy coding composes with LZ; it does not replace it. DEFLATE and zstd both run LZ first, then entropy coding.
