# PCM Fundamentals

PCM represents a waveform as evenly spaced amplitude samples.

- **Sample rate:** samples per second; the Nyquist limit is half the rate. 48 kHz is common for games; 44.1 kHz is common for CDs.
- **Bit depth:** dynamic range for integer formats; 16-bit PCM offers about 96 dB. Float32 is conventionally centered in `[-1,1]`.
- **Channels:** concurrent streams such as L/R or 5.1.
- **Frame:** one sample per channel at one instant. A stereo frame contains two samples.

One second of stereo float32 at 48 kHz is `48,000 × 2 × 4 = 384,000` bytes; 16-bit PCM uses 192,000 bytes. 48 kHz represents up to 24 kHz. The approximate 16-bit range is `20 log₁₀(65,536) = 96.3 dB`.

Do not confuse an audio frame with a video frame. Float32 is useful internally because mixing can exceed ±1 without losing information until final output; it does not add resolution at the speaker. Most device APIs and WAV files are interleaved (`L,R,L,R`), while some systems use planar channels.
