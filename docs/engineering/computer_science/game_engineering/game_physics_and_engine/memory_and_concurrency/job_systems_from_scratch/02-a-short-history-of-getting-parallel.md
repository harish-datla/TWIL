# A Short History of Getting Parallel


Why do Rust's Rayon and Java's ForkJoinPool still schedule work with an idea from a 1995 MIT research language? Because the design most engines use today took shape over about three decades, and most of its pieces came from research systems and shipped engines that came before.
Timeline: from Cilk (1995) to C++26

Each step assumed the ones before it: the work-stealing theory behind Cilk (1994–95)[20], the Chase-Lev deque (2005)[6], the data-layout discipline (2009), shipped engine designs from Naughty Dog and Bungie (2015)[2], and a standard-library vocabulary (2024).

[Back to Job Systems from Scratch](index.md)
