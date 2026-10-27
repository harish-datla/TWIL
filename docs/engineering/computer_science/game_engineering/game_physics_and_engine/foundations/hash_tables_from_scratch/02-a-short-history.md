# A Short History


Hashing is nearly as old as the stored-program computer: the first descriptions come from IBM in 1953. Two things from its history matter in practice. The core analysis (how many slots a lookup checks as the table fills) dates from Knuth in the 1960s and still holds, so the formulas in §5 are not new. And the designs engines ship today, Swiss tables and their relatives, are from 2017 on: they leave the theory alone and cut the constant factors with SIMD and cache-aware layouts.
Timeline: Luhn's 1953 memo to wyhash

[Back to Hash Tables from Scratch](index.md)
