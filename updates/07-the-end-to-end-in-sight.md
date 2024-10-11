---
published: Jan 05, 2024
author: [team]
---

# #7: The end(-to-end) in sight

We have verified all of the "zero-order" functions but for `main`.

## Next up

Our goal for the next iteration (to 20th) is to implement the code generation
phase of Xr0. Our reason for this detour from verifying the Lex program is that
in verifying the parser we have found ourselves making significant refactors to
the code with which we are no longer sure that the program still functions as
intended. Generating a working program will allow us to test end to end.

Goal:

1. Implement C code generation
   [[#19](https://todo.sr.ht/~lbnz/xr0/19)]

Stretch goals:

1. Simplify map and array data structures
   [[#13](https://todo.sr.ht/~lbnz/xr0/13)]

2. Write a first-version of the "topological sort" mechanism
   [[#16](https://todo.sr.ht/~lbnz/xr0/16)].
