# Radix Sort: Linear Time, No Comparisons


The draw-call keys from §1 are plain 32- or 64-bit integers, so sorting them doesn't need comparisons at all: you can sort by their bytes directly. Radix sort is a strong answer for "I have a million 32-bit keys and I want them sorted." It runs in O(d · n) where d is the number of digits (bytes, or whatever radix you pick), reads its input in sequential streams, and parallelizes well across cores or GPU threads. The catch is that keys must break into digits: fixed-width integers, floats reinterpreted as integers, or strings (with the MSD variant). Many renderers use it for draw-call sorting (§16).

Two flavors. LSD radix sort is the usual choice for fixed-width numeric keys: sort by the low byte first, then the next byte, and so on. Each pass is a stable counting sort into 256 buckets (for a byte radix). After four passes on 32-bit keys the array is fully sorted, because each pass is stable: keys that tie on this pass's byte keep the order the earlier, lower-byte passes gave them. MSD radix sort is the variant for strings and variable-length keys.

Worked example

Sort the 16-bit keys 0x0201, 0x0102, 0x0101, 0x0202 one byte at a time.

Pass 1, low byte: the keys ending 01 go first, in their original order (0x0201, 0x0101), then those ending 02 (0x0102, 0x0202).

Pass 2, high byte: high byte 01 first, keeping pass 1's order (0x0101, 0x0102), then 02 (0x0201, 0x0202). Sorted, with no comparisons between keys.
LSD radix sort on uint32 · C++

void radix_sort_u32(uint32_t* keys, uint32_t* scratch, int count) {
  uint32_t* src = keys;
  uint32_t* dst = scratch;
  for (int passShift = 0; passShift < 32; passShift += 8) {
    int bucketCount[256] = {0};
    // Pass 1: count how many keys go in each bucket.
    for (int i = 0; i < count; ++i)
      ++bucketCount[(src[i] >> passShift) & 0xFF];
    // Prefix sum: turn counts into starting offsets.
    int total = 0;
    for (int digit = 0; digit < 256; ++digit) {
      int keysWithDigit = bucketCount[digit];
      bucketCount[digit] = total;
      total += keysWithDigit;
    }
    // Pass 2: scatter keys into dst at their bucket offsets. Scanning src in
    // order keeps equal digits in order, which is what makes LSD radix correct.
    for (int i = 0; i < count; ++i) {
      int bucket = (src[i] >> passShift) & 0xFF;
      dst[bucketCount[bucket]++] = src[i];
    }
    std::swap(src, dst);   // The next pass reads what this pass wrote.
  }
  // Four passes is an even number of swaps, so the result is already back in
  // keys; the copy only matters if the pass count is changed to an odd number.
  if (src != keys) std::memcpy(keys, src, count * sizeof(uint32_t));
}

What's intentionally missing

Production radix sorts add what this listing skips: a payload (you sort key+index pairs or key+value structs, not bare keys); a skip check per pass (if one bucket holds all n elements the byte is constant and the whole pass can be skipped, common for the high bytes of small keys); the float sign-flip from the next section; and per-thread histograms for the multicore and GPU versions.

Cost: each pass reads the array once to build the histogram, then reads it again and writes it once to scatter, so roughly 3n memory operations per pass and ~12n across the four passes (count the loops in the listing above). The reads are sequential; the scatter writes go to 256 separate streams, which caches and write-combining handle well at this width. If memory bandwidth is the limit, 48 bytes of traffic per 4-byte key at 16 GB/s is roughly 3 ns/key for the whole sort. A comparison sort of a million keys typically costs tens of nanoseconds per key. For small arrays the fixed setup (four 256-entry histograms, four passes regardless of n) hands the win back to the comparison sort; published radix sorts such as ska_sort switch to std::sort below about 128 elements.
Live · LSD radix sort, byte by byte
elements 16
radix bits per pass
Run ▶
Reset
passes
···
total ops
···
vs n log n
···
The top row is the array before the current pass and the bottom row after it, colored by the digit this pass sorts on. Press Run, then switch between 4-bit and 8-bit digits; at the default 16 elements and 4-bit digits the readouts show 4 passes and 128 operations. The widget sorts 16-bit keys so each pass stays readable. Digit width is a tunable: more bits per pass means fewer passes but larger histograms, which cost setup time and cache space. 8 bits is the most common choice (Bevy's radsort crate sorts byte by byte), and 11-bit digits, three passes for a 32-bit key, are a well-known alternative; 4 bits is used here so the bucket structure is visible. "Total ops" counts one histogram read and one scatter write per element per pass.
Sorting floats with a radix sort

Radix sort handles depth keys because non-negative IEEE 754 floats, reinterpreted as 32-bit unsigned integers, sort in the same order as the floats. Negative values need a fix-up of a few lines: flip every bit of a negative float and only the sign bit of a non-negative one, and the whole range sorts correctly.

Worked example

1.0 is 0x3F800000; flip the sign bit and it becomes 0xBF800000. −1.0 is 0xBF800000; flip every bit and it becomes 0x407FFFFF. −2.0 is 0xC0000000, which becomes 0x3FFFFFFF.

As unsigned integers: 0x3FFFFFFF < 0x407FFFFF < 0xBF800000, the same order as −2 < −1 < 1.

Aras Pranckevičius's "Rough sorting by depth" post is a widely cited write-up of using the top bits of a float as a packed depth field[26]: since larger positive floats give larger integers, the top bits drop straight into a sort key. He keeps the top 10 bits; the sign bit is always zero for positive depths, so 9 of them vary, which works out to about seven buckets per factor of ten in distance (0.01 maps to 240, 1000 to 273). That coarseness is deliberate: front-to-back ordering of opaque draws only needs to be rough, since a misordered draw costs some overdraw, and coarse buckets leave room to sort by render state inside each one. Transparent draws that must blend in order need many more depth bits.

[Back to Sorting Algorithms](index.md)
