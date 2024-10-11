---
published: Dec 02, 2023
author: [team]
---

# #2: The matrix program verifies

We got the [matrix program] working.

  [matrix program]: https://github.com/xr0-org/xr0/blob/700ba6d668bef7d9adace6f537c871095336c4a5/tests/3-program/matrix.x

## What it took

- Our model is currently asymmetrical in the way it represents `alloc` and
  `dealloc` operations, meaning that `dealloc` statements can only be verified
  after an `alloc` statement has been executed. This poses a problem for
  verifying "destructors", i.e. functions that free memory given to them in an
  argument, because we have no way of indicating the assumption of previous
  allocation. To work around this we've introduced a kind of precondition
  capability, in which one labels "set-up" statements in the abstract with the
  tag `pre`; these are then executed in the verification process prior to either
  the body of the function or its abstract (during the abstract verification
  phase).

- We've been running into trouble when verifying functions like
	
    ```
    struct matrix *
    matrix_create(unsigned int rows, unsigned int cols);
    ```

  because we can't be sure that `rows` and `cols` are nonzero. This means that
  if we do a range of allocations over the interval `0:rows`, it is possible that
  this interval is empty, in which case no allocations would take place.  This
  creates a problem during deallocation (of the range) because until now our
  logic has involved searching for the objects (in the block containing the
  range) that include the lower- (here `0`) and upper-bound (`rows`) of the
  range, and then resizing the objects at the bounds and deallocating all the
  ones in between them. If `0:rows`, however, proves to be empty, then neither
  `0` nor `rows` will be in it, invalidating our approach despite the propriety
  of "recognising" that all the allocated memory in this potentially-empty range
  has been freed.

  As a hack we've hardcoded in the case where there is only one object
  (`0:rows`) in the block we're analysing, which happens to be the only case
  that obtains for our `matrix_destroy` currently, and this allowed us to close
  off the iteration.

- There's one or two (or 55kB worth of) leaks.
  
  ![Popeye Leaks](/updates/popeye-leaks.gif)

  If only there was some kind of tool for this :)

### Next up

The work on Xr0 continues.

Our goal for the next couple of months is to get a more sizeable program
verified with Xr0, and we'll be sharing more about this soon.

The coming iteration (to 16th) will be focused on refactoring and attempting to
refine the codebase, isolating the "essence" of what is required to verify the
matrix program. We regard this as a necessary next step because the codebase is
now at 6800 LOC and a lot of hacks (and leaks) have accrued, not to mention
confusion across our layers of abstraction.

Our goals for this iteration:

1. We will eliminate at least 75% of the leaks
   [[#4](https://todo.sr.ht/~lbnz/xr0/4)]

2. We will eliminate `EXPR_ACCESS` converging to pointer-based representations
   [[#5](https://todo.sr.ht/~lbnz/xr0/5)]

3. The concepts of a `value` and `variable` are very similar in our model:
   we will attempt to converge them
   [[#6](https://todo.sr.ht/~lbnz/xr0/6)]

4. We will make the execution paths involving lvalues and rvalues consistent and
   remove `resolve_bounds`
   [[#7](https://todo.sr.ht/~lbnz/xr0/7)]

5. We will refactor the `state` to remove the re-declarations of its
   submodules in its header file
   [[#8](https://todo.sr.ht/~lbnz/xr0/8)]

6. We will investigate a symmetric allocation/deallocation paradigm for at
   least three days
   [[#9](https://todo.sr.ht/~lbnz/xr0/9)].
