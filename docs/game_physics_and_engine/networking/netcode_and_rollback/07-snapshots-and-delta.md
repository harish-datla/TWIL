# Snapshots & delta

A 64-player match has hundreds of moving objects, and most of them didn't change since the last packet. Sending the whole world 20 times a second to every client wastes most of the bandwidth. A client-server engine takes a snapshot of world state and sends each client the delta against the last snapshot that client acknowledged[10]. The snapshot rate can sit below the tick rate: Counter-Strike: Source and TF2 simulate at 66 ticks/s, but Source clients request 20 snapshots/s by default (cl_updaterate 20)[9]. Quake 3 keeps the last 32 snapshots per client and deltas against the client's last acked one; if none is acked (heavy loss), it deltas against a zeroed baseline, which is just a full update. A lost ack self-heals: the next delta is computed from an older baseline (bigger, but correct) rather than a forced full resend.
Quantize and bit-pack the fields

Then quantize each field to the bits it needs (cross-ref Bit Shifting and Compression): map a position float over a known range to an integer (Fiedler's example packs x,y,z into 18/18/14 bits, ~2 mm precision, instead of 96) and orientation into a smallest-three quaternion (2 bits for the largest-component index + 3×9 bits = 29 bits vs 128)[11]. Quantization is lossy but bounded: picking the range and precision is the design choice, and out-of-range values clamp.

Worked example

At 1/512 m steps (about 2 mm), 18 bits give 218 = 262,144 steps, which covers 262,144 / 512 = 512 m; 14 bits cover 16,384 / 512 = 32 m, enough for height. That's 18 + 18 + 14 = 50 bits for a position instead of 96 for three floats, and 29 bits for an orientation instead of 128, so 79 bits per object instead of 224, about 35% of the size, before delta encoding drops the objects that didn't move at all.
