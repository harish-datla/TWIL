# Where embeddings mislead
Normalize before an inner-product index, or get a different metric

An index built for inner products returns the stored vector with the largest dot product, which favours long vectors. If cosine similarity is what the application means, scale every vector to length 1 before adding it and every query before searching, as the Faiss documentation specifies.[2]

"Similar" means whatever the training pairs made it mean. Co-occurrence embeddings put interchangeable words together, which includes antonyms and the barn–garage pair in the first widget. Pair-trained retrieval embeddings put questions near their answers. Pick a model by what its training objective rewarded, and judge it by its recall on a labelled retrieval set. Eyeballing cosines can't tell you which model retrieves better.

An approximate index trades recall for speed through ef. Report the recall at the chosen ef alongside any retrieval result, measured against brute force on a sample of queries as the sixteen-dimensional widget does. A system tuned for throughput can quietly drop the right passage one time in ten.

Vectors from two different models, or from two versions of one model, share no coordinate system: coordinate 17 in one means nothing in the other. Re-embed everything when the model changes, and never mix vectors from different models in one index.
