# Why neural networks?

Say you need code that decides whether a photo contains a cat. You can't write if (pixels look cat-shaped): no hand-written rule survives kittens, hairless breeds, plush toys and odd camera angles. What you can do is collect a few thousand photos labeled "cat" or "not cat", write a flexible function with millions of adjustable numbers, and let an algorithm adjust those numbers until the function labels the photos correctly. The rules are fitted to examples instead of programmed.

A neural network is that flexible function, and its adjustable numbers are called parameters. Training is the algorithm that adjusts them. A network trained this way can also label photos it has never seen, though it doesn't have to, and §14 shows how to check. The page builds each piece in order: a neuron and layers of neurons (§2 to §5), a loss that scores how wrong the network is (§6), gradient descent and backpropagation to shrink that score (§7 to §11), then a complete classifier you train yourself (§12 to §15).
Where the idea came from

