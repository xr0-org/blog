---
published: Jan 20, 2024
author: [team]
---

# #10: Topological sort and a speedbump

We failed to verify all the functions in parse.x, getting through all but four
functions. We did, finally, implement the topological sorting[^sorting] feature
that orders the functions in a translation unit[^translation] for verification.

  [^sorting]: [Topological sorting](https://en.wikipedia.org/wiki/Topological_sorting)
  where we regard functions as nodes in a graph and calls between them as
  vertices.

  [^translation]: A translation unit is a C file after its been preprocessed,
  see [here](https://xr0blog.substack.com/i/140569893/linkage-in-c).

The hurdle we encountered is a case where the value of one of the fields of a
structure returned from a function call determines whether or not an allocation
will take place, as exhibited in [this test]. We will go into more detail
regarding this in a future article.

  [this test]: https://github.com/xr0-org/xr0/blob/20a55036a5f056885f860254e8eee0e68eaa80eb/tests/1-branches/1000-chaining-functions.x#L28

## Next up

The goal for the next week (to 27th) is to get the test linked above to work
[#24](https://github.com/xr0-org/xr0/issues/24).

Stretch goal:

- Simplify map and array data structures [#13](https://todo.sr.ht/~lbnz/xr0/13).
