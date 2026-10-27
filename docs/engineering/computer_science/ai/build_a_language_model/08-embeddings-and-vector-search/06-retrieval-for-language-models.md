# Retrieval for language models

A language model knows what was in its training data, up to its cutoff date. Ask it about last week's release notes or your company's internal wiki and it has nothing to go on. The standard fix is retrieval: look up relevant text and put it in the prompt.[12]

    Split the reference documents into chunks.
    Embed each chunk and store the vectors in an index.
    When a question arrives, embed it and find the most similar chunks.
    Put those chunks into the prompt alongside the question.

Worked example

Question: "How do I reset my router?" Best passage: "Hold the button on the back for ten seconds." Words in common: zero, so a pure keyword search gives this passage no credit. An embedding model trained on (question, answer) pairs puts the two near each other anyway, because that's the relation its training pairs taught it.

Embedding search suits questions for exactly that reason: a question and its answer often share few words, which defeats keyword matching.[12] Each piece of this page has a job in that system. The embedding model is commonly trained with the in-batch contrastive loss from section 3 on (question, passage) pairs. The store is a flat or HNSW index over normalized vectors. The quality settings are the chunk size, the number of chunks retrieved, the index's recall at the chosen ef, and whether a keyword index is searched alongside and the results merged, which the cookbook lists among the ways to improve on embeddings-only search.[12] The fine-tuning page covers the other way to change what a model produces: training its weights further.
Check Your Understanding Q1 of 2

An HNSW index returns the true nearest neighbour for 90% of queries. Which single change most directly raises that to 99%?
Increase ef at query time.
Normalize the vectors.
Lower M to make the graph sparser.
0/2 correct
