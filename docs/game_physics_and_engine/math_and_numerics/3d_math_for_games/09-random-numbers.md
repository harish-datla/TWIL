# Random Numbers


Procedural placement, loot rolls, particle jitter, AI variation: games lean on random numbers, and the C standard library's rand() is the wrong tool for all of them.

rand() is implementation-defined: there's no required algorithm and no quality guarantee. Common implementations are LCGs (linear congruential generators, which compute state = a·state + c, mod m) whose low-order bits are weak, and RAND_MAX can be as small as 32767[15]. On top of that, rand() % n introduces modulo bias unless n divides RAND_MAX + 1[15].

Worked example

With RAND_MAX = 32767, rand() has 32,768 possible values. For rand() % 10, 32,768 = 3,276 × 10 + 8, so 0 to 7 each come from 3,277 values and 8 and 9 from 3,276: a bias you'd never see.

For rand() % 20000, 32,768 = 1 × 20,000 + 12,768, so results 0 to 12,767 each come from two values and 12,768 to 19,999 from one. The lower 64% of the range is twice as likely as the rest: a loot table indexed this way drops its first entries twice as often.

Reach for a modern small-state generator. PCG applies a permutation to an LCG's output: small fast state, strong statistical quality, multiple independent streams[14]. The xoshiro/xoroshiro family is comparably fast with tiny state. Both are plenty for gameplay.
Three things to get right

    For a bounded range, use rejection sampling or Lemire's multiply-shift method (which also rejects, rarely), not %, or you reintroduce bias.
    These are not cryptographically secure. Fine for loot and particles; never for anti-cheat seeds or security.
    Reproducible replays need a fixed seed and a fixed algorithm. That's another reason to avoid rand(), whose algorithm varies across platforms and compilers.

PCG32, a usable generator in a dozen lines
C++
Rust

// PCG32 (O'Neill). State is 64 bits; output is a permuted 32-bit word.
struct Pcg32 {
    uint64_t state = 0x853c49e6748fea9bULL;
    uint64_t inc   = 0xda3e39cb94b95bdbULL;   // stream selector, must be odd

    uint32_t next() {
        uint64_t oldState = state;
        state = oldState * 6364136223846793005ULL + inc;   // LCG step
        // Output uses the OLD state, so the permutation overlaps the next step.
        uint32_t xorshifted = (uint32_t)(((oldState >> 18) ^ oldState) >> 27);
        uint32_t rotation = (uint32_t)(oldState >> 59);   // top 5 bits pick the rotate
        // Rotate right by `rotation`; (32 - rotation) & 31 keeps the shift below 32.
        return (xorshifted >> rotation) | (xorshifted << ((32 - rotation) & 31));
    }
};

Worked example

Seeded with the constants above, both listings return 0x152ca78d, 0x027c6003 and 0xcb07bbf3 on their first three calls. A port that prints anything else has a shift, the rotate or the output state wrong; the output must come from the old state, before the LCG step.
What's intentionally missing

These generators show the core step only. A shipping RNG adds: unbiased bounded sampling (rejection or Lemire), a documented seeding routine, float generation in [0, 1), and a seek/stream API for reproducible replays. Use the library versions (rand_pcg in Rust, the reference PCG headers in C++) rather than hand-rolling those parts.

[Back to 3D Math for Games](index.md)
