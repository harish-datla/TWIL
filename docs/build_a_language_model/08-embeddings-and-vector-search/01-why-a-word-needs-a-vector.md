# Why a word needs a vector

Someone types "kitten food" into a search box. A useful system ranks a page about cat food above one about car parts, so the computer needs a numeric way to say that "kitten" and "cat" are close. Language models and vector search engines start the same way: they replace each word or document with a list of numbers, a vector, chosen so that related items get similar vectors.

The obvious encoding is a one-hot vector: number the words in the vocabulary, and represent word number i as a row of zeros with a single 1 in position i. It's easy to build and useless for similarity. Two different words never share a position, so every pair of words is the same distance apart and nothing about meaning can be read off the vectors.[1]

Worked example

Vocabulary: cat, dog, car. One-hot: cat = (1, 0, 0), dog = (0, 1, 0), car = (0, 0, 1). The dot product of any two is 0, and every pair is √2 ≈ 1.41 apart.

Now give each word two made-up numbers instead: cat = (0.9, 0.1), dog = (0.8, 0.3), car = (0.1, 0.9). The cosine similarity of cat and dog is 0.97; of cat and car, 0.22. This encoding puts cat next to dog. The one-hot encoding can't.

An embedding is that second kind of vector: short, with most entries non-zero, and learned from data so that similar items end up pointing in similar directions. A one-hot vector needs one entry per vocabulary word, tens of thousands for a typical language model; an embedding typically has a few hundred. A transformer's first layer is a table of these vectors with one row per token, and a search engine's index is a table of them with one row per document.

There are two common ways to compare embeddings. The dot product multiplies matching entries and adds them up; it's large when two vectors point the same way, and it also grows with their lengths. Cosine similarity divides the dot product by both lengths, so it compares direction only and runs from −1 to 1. Faiss, a widely used vector-search library, has no separate cosine setting: you scale every vector to length 1 before indexing it and before searching with it, and after that the dot product and the cosine are the same number.[2]

Worked example

(3, 4) has length √(9 + 16) = 5. Divided by 5 it becomes (0.6, 0.8), which has length 1. For two vectors of length 1, the cosine is the dot product divided by 1 × 1, so the two scores match exactly.
