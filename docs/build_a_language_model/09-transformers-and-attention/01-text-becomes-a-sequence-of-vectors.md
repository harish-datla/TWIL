# Text becomes a sequence of vectors

A model can't read characters. It needs numbers, and it needs a fixed, finite set of pieces to assign them to. The tokenizer does that job: it splits text into tokens, the pieces in the model's vocabulary, and replaces each with an integer ID. A token can be a whole word, a fragment of one, a single character or a byte sequence.

Each ID then picks one row from a learned embedding table: a list of numbers that training arranges so that related tokens get similar vectors. The Embeddings page shows how such vectors are learned. From here on, the text is a sequence of vectors, one per token.

Where do the pieces come from? BPE (byte pair encoding) is one common way to learn them from data. In Sennrich et al.'s character-based version, each word starts as single characters plus an end-of-word marker. Count every pair of neighbouring symbols across the training words, weighted by how often each word appears. Merge the most frequent pair into one new symbol, then count again and repeat. The list of merges, in order, becomes part of the tokenizer.[1]

Worked example

Training words: low, low, lower, newest, widest. They start as l o w </w> (twice), l o w e r </w>, n e w e s t </w> and w i d e s t </w>: 28 symbols in all.

The pairs (l, o) and (o, w) each appear 3 times, more than any other pair. A tie-break rule picks (l, o), so every "l o" becomes "lo", leaving 25 symbols. Recount: (lo, w) now appears 3 times, so the second merge makes "low". After two merges "low" is a single token and "lower" is low e r </w>.
Learn adjacent-symbol merges
Toy corpus
Step
Play
Reset
0 learned merges; 28 corpus tokens including word ends. Start from characters and a word-end marker.

Click Step to learn one merge, or Play to run them all, and read each word's current split. The marker </w> keeps word endings distinguishable, so "est" at the end of a word becomes a different symbol from "est" inside one. Repeated words count more heavily. Pair counts are recomputed after every merge, so the next winning pair can change, and ties are broken by a fixed alphabetical rule. On the default corpus the first two merges build "low" and the next three build "est</w>". This is character-based BPE on a tiny vocabulary, not a byte-level production tokenizer. Reset removes the learned merges.

The widget shows training. Tokenizing new text later doesn't relearn anything: it applies the stored merges, in their learned order, to the new words.

The integer ID itself carries no meaning. Token 80 isn't "twice" token 40; the IDs are just row numbers. Meaning lives in the embedding vectors, and later layers adjust those vectors using the surrounding text, so the same token can end up with different hidden representations (the vectors the model carries for each position) in different sentences.

Tokenization is part of the model's contract. Feed a model IDs from a different vocabulary or merge table and it reads the wrong embedding rows. Context limits are counted in tokens, not characters, and the same number of characters can take very different numbers of tokens depending on the language and content, so a character budget doesn't guarantee a token budget.
