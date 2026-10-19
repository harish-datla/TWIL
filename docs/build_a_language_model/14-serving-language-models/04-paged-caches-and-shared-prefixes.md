# Paged caches and shared prefixes

Continuous batching needs a cache that grows and shrinks per conversation, and the server doesn't know how long a conversation will get when it starts. Reserving the maximum length for every slot up front wastes whatever a conversation doesn't use, and one that ends early leaves most of its reservation empty.

PagedAttention borrows the operating system's answer: split memory into fixed-size pages. Each conversation's cache lives in blocks of a fixed number of tokens (16 in the example in vLLM's design document), found through a per-conversation block table. A conversation takes a new block only when its last one fills, so the only waste is the unfilled tail of that last block.[12]

Worked example

A cache budget of 32,768 token slots, and a 300-token conversation.

Reserve 2,048 per conversation: only 32,768 / 2,048 = 16 fit, and this one uses 300 of its 2,048 slots, about 15%.

Blocks of 16: it takes 19 blocks, 304 slots, wasting 4.

Blocks also make sharing possible. Many requests start with the same system prompt, and its keys and values come out the same each time. vLLM gives each full block a hash of its tokens combined with the hash of the block before it, so a block's hash identifies the whole prefix up to that point. When a new request's block hashes match blocks already in memory, its block table points at the existing blocks. A reference count per block tracks how many conversations use it. Freed blocks go to a queue ordered by last use and keep their contents, and when memory runs short, the least recently used are evicted first.[13]

Reuse saves the prefill work for the shared prefix as well as its memory. llama.cpp's server likewise skips re-processing a prompt prefix it already holds in a slot's cache.[8]
Reserve per sequence or allocate blocks on demand
Block size
Reserved length
Shared prefix
Seed
1
Reset
Budget of 32,768 token slots; 64 candidate sequences of 150 to 600 tokens each. Reserving 2048 tokens per sequence fits 16 sequences and uses 19% of what it reserves. Blocks of 16 fit 64 sequences in 1515 blocks (0 shared prefix blocks), 98% full; 445 token slots lost to partial last blocks.

The left panel counts the sequences that fit in a budget of 32,768 token slots under each scheme; the right compares slots used with slots allocated. Change the block size, reserved length and shared prefix. 64 candidate sequences of 150 to 600 tokens compete for the budget. Reserving 2,048 per sequence admits 16 and fills 19% of what it reserves. Blocks of 16 admit all 64 at 98% occupancy, losing 445 slots to partly filled last blocks. Raise the reservation to 8,192, the sequence length Llama 3 was trained on, and only 4 fit.[4] Larger blocks waste more at the tail: 256-token blocks are 76% full, against 98% for the 16-token blocks of vLLM's design document.[12] A 1,024-token shared prefix is stored once, in 64 blocks every sequence's table points at, so the paged count doesn't move. The reserved scheme can't share; its occupancy rises to 69% only because each of its 16 sequences stores its own copy of the prefix.
