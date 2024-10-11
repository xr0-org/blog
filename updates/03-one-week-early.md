---
published: Dec 09, 2023
author: [team]
---

# #3: One week early

Whether we overachieved or under-attempted, the reader must judge, but we are
done with the items we laid out last week.

## What it took

We did (1.), (2.), (4.) and (5.) as they were stated.

For (3.) we concluded that the concepts of value and variable are sufficiently
different to warrant their remaining as such in our model for now. We use
`variable` to represent the variables which are "within scope" on a given line of
code (this would include local variables, parameters and globals), but value
represents what can be placed in a given location in memory.

For (6.) we investigated and found that it is likely possible to implement a
symmetric paradigm for allocation/deallocation. We were interested in this
paradigm over against what we currently have (in which deallocations basically
delete allocated blocks) because it would allow us to describe and verify
functions that deallocate memory without having to first allocate memory. In
this way our abstract descriptions of what such "deconstructors" would do would
focus on the (attempted) operations in the body, without worrying about the
conditions under which they can be safely/effectively executed. We found,
however, that using the _setups_ (what we're calling the `pre`-labelled
statements in the abstract) that we had introduced in our previous iteration
we're able to verify deconstructive functions in a fairly minimal way, so it
seems premature to focus on minifying or eliminating setups altogether.

The main caveat here is we aren't sure about the logical of verifying functions
by relying on setups. In particular, it may be possible to construct a function
that verifies using certain setups but doesn't perform the operations specified
in its abstract. This would mean that at _simulation-time_ (i.e. when the
function is called) the effects it introduces would differ from what is checked
during its verification.

## Next up

For the next week we'll continue focusing on refactoring the codebase.

Goals:

1. Move the functionality of the verify module into the ast module
   [[#10](https://todo.sr.ht/~lbnz/xr0/10)]

2. Refactor the state module in light of what we learn in (1.)
   [[#11](https://todo.sr.ht/~lbnz/xr0/11)]

3. Write up the next program to be verified
   [[#12](https://todo.sr.ht/~lbnz/xr0/12)].

Stretch goals:

1. Simplify map and array data structures
   [[#13](https://todo.sr.ht/~lbnz/xr0/13)]

2. Convert `EXPR_MEMORY` into an `ast_stmt`
   [[#14](https://todo.sr.ht/~lbnz/xr0/14)].
