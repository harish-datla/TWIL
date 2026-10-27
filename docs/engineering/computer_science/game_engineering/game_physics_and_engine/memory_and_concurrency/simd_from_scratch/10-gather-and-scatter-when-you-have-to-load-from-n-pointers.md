# Gather and Scatter: When You Have to Load from N Pointers


Skinning reads bone data for every vertex, and each vertex names its bones by index, so eight vertices want data from eight unrelated addresses. A vector load assumes contiguous addresses. A gather instruction (AVX2 and later) takes a base pointer and a vector of indices, and loads one element per lane from base[indices[lane]]. The hardware does N independent loads internally and assembles the result:
gather.cpp · per-vertex skinning indirection

// 8 vertices, each with a bone-index. Look up the 8 bone matrices' first entries.
float bonePositionX[MAX_BONES];        // SoA bone data, one float per bone
int boneIndices[8] = {3, 17, 3, 8, 42, 8, 17, 5};

__m256i indicesVector = _mm256_loadu_si256((__m256i*)boneIndices);
// Gather: each lane loads bonePositionX[indicesVector[lane]], 4-byte stride.
__m256 boneXVector = _mm256_i32gather_ps(bonePositionX, indicesVector, 4);

Gather is the direct way to vectorize random-access patterns: skinning with per-vertex bone indices, lookup-table-driven shading, ECS systems where the components for a job aren't laid out in the order the job processes entities. The cost picture is the part to understand.

Gather has both a latency and a throughput number, and they are very different. On Haswell (2013, the first AVX2 part), a vpgatherdd ymm (8 × int32) takes around 20 cycles to produce its result and decodes to 34 μops, with a reciprocal throughput of 11 cycles[1]: a back-to-back stream of gathers starts a new one every 11 cycles. On Haswell, eight scalar loads plus inserts often matched or beat it. Skylake cut the reciprocal throughput to 5 cycles, Ice Lake stayed at 5, and Alder Lake's P-cores reach about 3. AMD's gathers were slower for longer: 16 cycles on Zen 2, 8 on Zen 3 and Zen 4, under 6 on Zen 5[1]. The numbers can also move after a chip ships: the August 2023 microcode mitigation for the Downfall vulnerability (a flaw that let gathers leak stale data from inside the CPU) slowed gathers on Skylake through Ice Lake / Tiger Lake client and server parts. Intel cited penalties of up to 50% in extreme cases; Phoronix's application benchmarks found smaller but still significant losses in gather-heavy code[39].

A useful rule of thumb: a gather costs about one load-port slot per lane, so its throughput ceiling is close to that of the same number of scalar loads; what it saves is the instructions to insert each value into a vector. It wins when the data is cache-resident and the alternative is eight scalar loads and seven inserts. It loses badly when the indices turn out to be sequential, where one ordinary vector load does the same job.

Worked example

The listing's indices, {3, 17, 3, 8, 42, 8, 17, 5}, name only 5 distinct bones. At 4 bytes per float they sit at byte offsets 12, 68, 12, 32, 168, 32, 68 and 20, which fall in 3 cache lines if the array starts on a 64-byte boundary.

By the rule of thumb above, the gather still costs about 8 load-port slots, one per lane. Sorted by bone, the same vertices would read their bones from a few contiguous runs instead.

Scatter (the reverse: write each lane to base[indices[lane]]) is AVX-512 only. When two lanes name the same destination, the element writes are architecturally ordered from the lowest lane to the highest, so the highest lane's value is the one that sticks; the hardware may skip the earlier, fully-overlapped writes[2]. Scatter is also slow: a 16-lane vpscatterdd has a reciprocal throughput of about 17 cycles on Skylake-X, roughly one element per cycle, and about 11 on Ice Lake and Alder Lake[1].

The pragmatic answer in most engine code is to avoid gather and scatter by restructuring the data:

    Sort by bone index before skinning; vertices with the same bone now load contiguously.
    Sort by component-id in an ECS so the inner loop touches one chunk's worth of components per iteration.
    If indices are stable across many frames, transpose once into a denser layout and reuse it.

When gather is unavoidable, the typical pattern is to gather a small amount of data (a single float per lane), then do many arithmetic ops in vector form before the next gather, so the gather's cost is spread over a lot of compute.

[Back to SIMD from Scratch](index.md)
