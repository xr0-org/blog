---
published: Jan 01, 2024
---

# #6: New year

Although this update comes two days late, we did succeed in achieving our goals
for the last iteration by the 30th (of last month).

## Next up

Our overarching aim for the next few iterations remains as verifying the parser
for Lex. In keeping with this, getting the "zero-order" allocating and
deallocating functions to verify seems to be the only possible next step along
the critical path for our aim, so we're returning to it as the goal for this
iteration (upto 6th).

Goal:

1. Verify all but one of the functions that directly invoke malloc (or calloc,
   realloc) and free
   [[#15](https://todo.sr.ht/~lbnz/xr0/15)]

Stretch goals:

1. Simplify map and array data structures
   [[#13](https://todo.sr.ht/~lbnz/xr0/13)]

2. Write a first-version of the "topological sort" mechanism
   [[#16](https://todo.sr.ht/~lbnz/xr0/16)].
