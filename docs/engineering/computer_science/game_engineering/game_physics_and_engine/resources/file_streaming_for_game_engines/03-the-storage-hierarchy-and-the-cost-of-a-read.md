# The Storage Hierarchy and the Cost of a Read


Your loader runs fine on the NVMe drive in your dev machine and takes minutes on a player's HDD, a hard disk with a spinning platter and a moving read head. Streaming design starts from where the data actually lives, because the latency gap between "in L1 cache" and "on a spinning disk" is seven orders of magnitude. Loading code tuned only on a fast dev machine can be unplayable on minimum-spec hardware.
Tier	Typical latency	Bandwidth	Time to read 64 KB (log scale)
L1 cache	~1 ns	~1 TB/s	
L3 cache	~10 ns	~500 GB/s	
DDR5 RAM	~80 ns	~70 GB/s	
NVMe Gen 5	~50 µs	~14 GB/s	
NVMe Gen 4	~70 µs	~7 GB/s	
SATA SSD	~150 µs	~550 MB/s	
HDD (sequential)	~10 ms (first seek)	~150 MB/s	
HDD (random 4 KB)	~13 ms per IO	~0.3 MB/s at QD1	

Sources: NVMe Gen 4 sequential throughput matches the Samsung 990 Pro datasheet (7,450 MB/s sequential read, 1.4M IOPS at QD32/16T)[13]; Gen 5 throughput matches the Crucial T705 review (~14.5 GB/s sequential)[14]. The cache, RAM and disk latencies follow Jeff Dean's widely quoted "Numbers Everyone Should Know"[15]. The HDD random row assumes a 7,200 rpm drive: about 8.5 ms of seek plus 4.2 ms of average rotational latency per read. The bar column is log-scaled: the span from L1 to an HDD random read is about six orders of magnitude, which no linear bar can show.

Two numbers matter independently: latency (how long any single read takes) and bandwidth (how much data per second you can sustain). A modern NVMe can serve 7 GB/s of sequential reads, but the first 4 KB still takes tens of microseconds; if you're scattering reads, latency dominates and you'll never see the bandwidth number on the box. Designing a streamer is mostly figuring out how to coalesce reads so the bandwidth number is the one that matters.

Worked example

Load a 1 MiB texture from the table's NVMe Gen 4 drive (about 70 µs latency, 7 GB/s). As one read it costs 70 µs plus 1 MiB ÷ 7 GB/s = 150 µs of transfer: about 220 µs. As 256 separate 4 KB reads issued one after another, each pays the full 70 µs plus under a microsecond of transfer: 256 × 70.6 µs ≈ 18 ms, more than a whole 60 fps frame and about 80 times slower for the same bytes.

The widget below races a fixed payload (320 MB by default) across four storage classes. Toggle the access pattern between sequential and random 4-KB reads at queue depth 32; the HDD's random bar collapses to a sliver:
Live · Storage Latency Racer
sequential
random 4 KB, QD32
payload size 320 MB
RACE ▶
Reset ⟲
HDD
2.13 s
SATA SSD
582 ms
NVMe Gen 4
46 ms
NVMe Gen 5
23 ms
Each bar is one drive reading the same payload; press Race, then switch to random reads and race again. At the default 320 MB the sequential times read 2.13 s for the HDD, 582 ms for the SATA SSD, 46 ms for NVMe Gen 4 and 23 ms for Gen 5. Random 4 KB reads leave the NVMe drives close behind (57 ms and 53 ms) but take the HDD to 266.67 s, about four and a half minutes: its run is dominated by per-IO seek time, even with 32 reads queued for the drive to reorder. The random rates assume that deep queue: at queue depth 1 (the naïve loader in §4) even a fast NVMe drops below 100 MB/s.
A trap engineers fall into

"NVMe is fast" is true but useless. You don't get 7 GB/s unless you submit large reads at high queue depth. A naïve loop that reads one 4-KB block at a time, blocking each time, sees under 100 MB/s on the same drive (the 990 Pro's queue-depth-1 rating works out to about 90 MB/s[13]). Most of the work in the rest of this tutorial is about keeping the queue full.
Check Your Understanding Q1 of 2

Your engine reads 100,000 small files from disk on startup. Each file is 8 KB. Loading takes 15 minutes on an HDD and 2 seconds on an NVMe. Why is the HDD so much slower than the bandwidth ratio would predict?
The HDD has less sequential bandwidth, and 800 MB of files takes minutes to transfer at HDD transfer rates.
Each 8-KB file forces a seek on the HDD, costing ~10 ms regardless of file size: 100,000 seeks × 10 ms is 1,000 seconds of raw seek time, which request reordering only partly claws back. The NVMe pays microseconds per command, not milliseconds.
The HDD’s sectors are smaller than the NVMe’s pages, so each 8-KB file costs more physical operations to assemble.
0/2 correct

[Back to File Streaming for Game Engines](index.md)
