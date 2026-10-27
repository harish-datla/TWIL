# DirectStorage, PS5, and the Modern Fast Paths


A PC game reads from the same class of NVMe drive as a PS5, but on Windows every read passes through the file system and a stack of filter drivers, then waits for a CPU decompressor, while a PS5 read goes through a custom I/O path with a hardware decompressor. Everything so far works on top of the OS's normal file API. On PC that was the only option until DirectStorage. The 2020 consoles (PS5 and Xbox Series) shipped with dedicated I/O and decompression silicon, and PC has spent the years since adding software and driver equivalents.
DirectStorage: what the API actually skips

Microsoft's DirectStorage (1.3 is the current full release; 1.4 has been in public preview since March 2026[12]) is the explicit PC fast path, built from a few specific decisions:

    Batched submission. Requests queue up in user mode (EnqueueRequest, or EnqueueRequests for a whole batch since 1.3), and one Submit hands the batch to the runtime, which amortizes the per-request overhead across the batch.
    BypassIO. A Windows 11 read path that sends a request from the I/O manager through NTFS (Windows' default file system) straight to the disk and NVMe drivers, skipping every file-system filter (antivirus and similar), volume-stack filter and storage-stack filter in between[33]. Currently client Windows only, NVMe-only, NTFS-only and noncached reads only, and any filter that hasn't opted in (BitLocker's, for example) sends reads fully or partly back down the traditional path.
    GPU-side decompression. A compute-shader decoder for GDeflate (since 1.1) and Zstd (1.4 preview) reads the compressed buffer in VRAM and writes the decompressed buffer to the target resource, in the same place the GPU was going to read it from. The CPU never sees the uncompressed bytes.
    Fence synchronization with D3D12. A request batch can signal a D3D12 fence (a counter the CPU or GPU can wait on) on completion, and since 1.3 it can also wait on one before running[11]. You can compose stream loads into your existing graphics queue without polling.

directstorage_sketch.cpp · loading a compressed texture

// ── One-time setup ────────────────────────────────────────────────────
// The factory is the entry point to the DirectStorage runtime.
ComPtr<IDStorageFactory> storageFactory;
DStorageGetFactory(IID_PPV_ARGS(&storageFactory));

// Size the staging buffer compressed bytes bounce through on the way to
// GPU decompression. The API takes a plain byte count; the default is
// 32 MiB (DSTORAGE_STAGING_BUFFER_SIZE_32MB), and Microsoft's 1.1 release
// benchmarks needed ~128 MiB to saturate the I/O stack.
storageFactory->SetStagingBufferSize(128 * 1024 * 1024);

// Create a "GPU queue": reads land directly in GPU resources, with
// optional GPU-side decompression on the way.
ComPtr<IDStorageQueue1> gpuQueue;
DSTORAGE_QUEUE_DESC queueDescriptor{};
queueDescriptor.SourceType = DSTORAGE_REQUEST_SOURCE_FILE;
queueDescriptor.Capacity   = DSTORAGE_MAX_QUEUE_CAPACITY;   // max outstanding requests
queueDescriptor.Priority   = DSTORAGE_PRIORITY_NORMAL;
queueDescriptor.Device     = d3d12Device.Get();          // the D3D12 device we'll write into
storageFactory->CreateQueue(&queueDescriptor, IID_PPV_ARGS(&gpuQueue));

// Open the asset bundle once. We'll keep this handle for the lifetime
// of the game and submit many reads against it.
ComPtr<IDStorageFile> bundleFile;
storageFactory->OpenFile(L"assets.pak", IID_PPV_ARGS(&bundleFile));

// ── Per asset: describe the read, push it into the queue ─────────────
DSTORAGE_REQUEST textureRequest{};

// Where the compressed bytes are coming from.
textureRequest.Options.SourceType        = DSTORAGE_REQUEST_SOURCE_FILE;
textureRequest.Source.File.Source        = bundleFile.Get();
textureRequest.Source.File.Offset        = manifestEntry.byteOffset;
textureRequest.Source.File.Size          = manifestEntry.compressedSize;

// How to decompress them. GDeflate is decoded by a GPU compute shader;
// the CPU never sees the uncompressed bytes.
textureRequest.Options.CompressionFormat = DSTORAGE_COMPRESSION_FORMAT_GDEFLATE;   // or DSTORAGE_COMPRESSION_FORMAT_ZSTD (1.4 preview)
textureRequest.UncompressedSize          = manifestEntry.uncompressedSize;

// Where the decompressed bytes end up: directly into a region of an
// existing D3D12 texture resource.
textureRequest.Options.DestinationType            = DSTORAGE_REQUEST_DESTINATION_TEXTURE_REGION;
textureRequest.Destination.Texture.Resource       = textureResource.Get();
textureRequest.Destination.Texture.SubresourceIndex = mipLevelIndex;
textureRequest.Destination.Texture.Region         = textureRegion;

gpuQueue->EnqueueRequest(&textureRequest);

// Submit the batch and signal a D3D12 fence when the GPU is finished
// writing. Any other GPU work that consumes the texture can wait on
// the same fence value, no CPU polling required.
gpuQueue->EnqueueSignal(streamingFence.Get(), nextFenceValue);
gpuQueue->Submit();

The shape mirrors io_uring: build a request, push it, eventually reap completions. The pieces that are new are which hardware sees the data and when: the compressed bytes go disk → staging buffer in system memory → copy to VRAM → compute-shader decode → final resource, and the CPU handles the request descriptors, not the data.
PS5 and Xbox Series: silicon shortcuts

The console architectures predate DirectStorage on PC and chose a different tradeoff. Both put dedicated decompression hardware on the I/O path, so the codec runs in fixed-function silicon rather than on a compute shader.

    PS5. Mark Cerny's "Road to PS5" describes a custom 12-channel NVMe interface, a dedicated DMA controller, two I/O coprocessors, and a hardware Oodle Kraken block[7]. Raw read: 5.5 GB/s. Effective post-Kraken: typically 8-9 GB/s. Oodle Texture's RDO (rate-distortion-optimized) encoding, which trades a little quality for a lot of compressibility, raises the ratio on texture data, which Bloom expects to bring the whole-game average closer to 2:1[25].
    Xbox Series. Microsoft's Velocity Architecture[8]: a 2.4 GB/s NVMe, a hardware LZ block, a hardware BCPack block (texture-format aware, more effective on BCn than general-purpose codecs), DirectStorage, and Sampler Feedback Streaming. Effective I/O performance: 4.8 GB/s at a 2:1 ratio. Microsoft describes the result as "approximately 100x the I/O performance in current generation consoles", meaning Xbox One.

Worked example

A batch of textures takes 100 MB on disk, compressed 2:1. On PS5 the drive reads it in 100 MB ÷ 5.5 GB/s ≈ 18 ms, and the Kraken block hands back 200 MB without a CPU core or a shader touching it. On Xbox Series at 2.4 GB/s raw the same read takes about 42 ms. On PC without DirectStorage, one core of zlib producing those 200 MB at 0.4 GB/s would need half a second.
Why two answers to the same question

Hardware decompression is faster per watt and frees up shader cores. Compute-shader decompression is more flexible and doesn't add silicon area. The console answer suits a fixed platform where the codec can be chosen once for the generation; the PC answer is what let GDeflate and then Zstd arrive as SDK and driver updates rather than new chips.

[Back to File Streaming for Game Engines](index.md)
