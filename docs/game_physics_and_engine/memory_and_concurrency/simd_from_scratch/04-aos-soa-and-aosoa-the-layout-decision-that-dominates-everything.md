# AoS, SoA, and AoSoA: The Layout Decision That Dominates Everything


A particle update reads the positions and velocities of 100,000 particles every frame. How those particles sit in memory decides whether the vector unit can load eight of them at once. A scalar Particle in an engine is often laid out as:
particle_aos.cpp · the natural object layout

struct Particle {     // 32 bytes per particle
  float positionX, positionY, positionZ;
  float velocityX, velocityY, velocityZ;
  float lifetime;
  float sizePixels;
};

Particle particles[100000];

// Update step. Scalar, one particle at a time.
void tickScalar(float deltaTime) {
  for (int i = 0; i < 100000; ++i) {
    particles[i].positionX += particles[i].velocityX * deltaTime;
    particles[i].positionY += particles[i].velocityY * deltaTime;
    particles[i].positionZ += particles[i].velocityZ * deltaTime;
    particles[i].lifetime  -= deltaTime;
  }
}

This is Array of Structures (AoS): one record at a time, every field of one particle contiguous. It is what a C++ programmer reaches for by default, and it is awkward to vectorize. To produce one SIMD register of positionX values, the CPU has to read positionX from particle 0, skip 28 bytes, read positionX from particle 1, skip 28 bytes, and so on. A 32-byte vector load at particle 0 picks up exactly one particle: one positionX and seven other fields. Filling a register with eight positionX values takes either eight separate loads or eight vector loads plus a transpose of roughly two dozen shuffles.

The cure is to transpose:
particle_soa.cpp · structure of arrays

struct ParticleSoa {
  float* positionX;   // N floats, contiguous
  float* positionY;
  float* positionZ;
  float* velocityX;
  float* velocityY;
  float* velocityZ;
  float* lifetime;
  float* sizePixels;
  int    particleCount;
};

// Update step, AVX2. Processes 8 particles per loop iteration.
// Assumes particleCount is a multiple of 8 (pad the arrays; §5 covers tails).
void tickVector(ParticleSoa& particles, float deltaTime) {
  __m256 deltaTimeVector = _mm256_set1_ps(deltaTime);  // broadcast scalar to all 8 lanes
  for (int i = 0; i < particles.particleCount; i += 8) {
    // One load per stream, one FMA per stream. Eight particles updated per iteration.
    __m256 posX = _mm256_loadu_ps(particles.positionX + i);
    __m256 velX = _mm256_loadu_ps(particles.velocityX + i);
    posX = _mm256_fmadd_ps(velX, deltaTimeVector, posX);          // posX += velX * dt
    _mm256_storeu_ps(particles.positionX + i, posX);

    __m256 posY = _mm256_loadu_ps(particles.positionY + i);
    __m256 velY = _mm256_loadu_ps(particles.velocityY + i);
    posY = _mm256_fmadd_ps(velY, deltaTimeVector, posY);
    _mm256_storeu_ps(particles.positionY + i, posY);

    __m256 posZ = _mm256_loadu_ps(particles.positionZ + i);
    __m256 velZ = _mm256_loadu_ps(particles.velocityZ + i);
    posZ = _mm256_fmadd_ps(velZ, deltaTimeVector, posZ);
    _mm256_storeu_ps(particles.positionZ + i, posZ);

    __m256 lifeRemaining = _mm256_loadu_ps(particles.lifetime + i);
    lifeRemaining = _mm256_sub_ps(lifeRemaining, deltaTimeVector);
    _mm256_storeu_ps(particles.lifetime + i, lifeRemaining);
  }
}

Same arithmetic, eight particles per iteration. Every load pulls in eight useful floats, half a cache line. Every FMA does eight multiply-adds, and no shuffles are needed to get the data into lanes.

The widget below shows what one load picks up for a pass that reads only positionX, such as a sort key or a coarse cull. In AoS the 32-byte load fetches one particle's worth of mixed fields, of which one is wanted; in SoA the same load fetches eight consecutive positionX values. The array scrolls past a fixed box; the cells inside the box are the bytes a single vmovups brings in.

Worked example

100,000 particles at 32 bytes each is 3.2 MB in AoS. A pass that reads only positionX still pulls all 3.2 MB through the cache, because cache lines arrive whole and each one holds every field of two particles.

In SoA the positionX array is 100,000 × 4 bytes = 400 KB, one-eighth as much. One 32-byte load shows the same ratio: 4 useful bytes in AoS against 32 in SoA.
Live · AoS vs SoA cache picture
AoS (object layout)
SoA (transposed)
The colored strip is memory scrolling under one 32-byte load, the white box. Toggle AoS and SoA and read the panel under the strip. For a pass that reads only positionX, each 32-byte load in AoS delivers 4 useful bytes; in SoA it delivers 32. Cache lines arrive whole, so the AoS version of that pass also pulls roughly 8× more data from DRAM[18], which is why the layout switch is often a bigger win than the SIMD switch itself. The particle tick above reads seven of the eight fields, so its memory traffic is similar in both layouts; its AoS cost is the shuffling needed to move each field into lanes.

The third option is AoSoA: an array of blocks, each holding a SIMD width's worth of records in SoA form. A block of 8 particles' xs, then 8 ys, then 8 zs, then on to the next 8 particles. This keeps the lane-friendly access pattern inside a block while keeping one record's fields within a few cache lines of each other. Archetype-based ECS frameworks (entity-component systems, which store game-object data as arrays of components) use the same idea at a larger grain: Unity's Entities package stores each component type as its own array inside fixed 16 KiB chunks[11], which is SoA inside a chunk and AoSoA across chunks. Mike Acton's data-oriented design talk makes the general case for choosing layout by access pattern[13].

Three rules to remember when picking a layout:

    SoA pays where a kernel touches few fields of many records. The particle tick uses position and velocity; the cull pass uses just position and boundsRadius. SoA wins decisively here because the unused fields don't get pulled into the cache at all.
    AoS pays where a kernel touches many fields of few records. Inserting an item into a sorted container, dereferencing a single picked entity for inspection, serializing one record to disk. Don't fight the layout for these.
    The decision is per-system, not per-codebase. A renderer that has one hot pass over many entities and occasional access to a few selected entities reasonably stores both: a SoA "transform pool" feeding the renderer, with a thin AoS handle for game logic. The translation lives at the system boundary and runs once per frame.

Check Your Understanding Q1 of 1

You have 100,000 entities. Two systems need access: a frequent tickPhysics() that touches position and velocity sixty times a second, and an occasional serializeForSave() that touches every field once per save (one save per minute). Which layout is right?
SoA, with serializeForSave doing a scatter-write back into a temporary AoS buffer at save time.
AoS, because the data is per-entity and that's the natural object-oriented layout.
AoSoA with block size 8.
0/1 correct

[Back to SIMD from Scratch](index.md)
