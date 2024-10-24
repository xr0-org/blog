---
published: Jun 08, 2024
author: [team]
---

# #16: Steps (rather than leaps) into bounds

## tl;dr

Work continues on [buffer overflows](https://github.com/xr0-org/xr0/milestone/1).
This week we’ve augmented Xr0’s declaration syntax and parsing to capture array
notation and we’re partway into implementing the basic semantics of
stack-arrays. Our goal for this week is to finish the array logic, as well as
the [simplest buffer case](https://github.com/xr0-org/xr0/issues/57), that where
both the accessed index and the size of the buffer are constants defined locally
in a function.

## Some details

Whenever a pointer is dereferenced, we must insist that the local context
establishes, not only (i) that the pointer corresponds to a valid block, but
that (ii) the operation takes place within the bounds of the corresponding
block[^block] We have substantially solved (i) in our work on the use-after-free (UAF)
problem and uninitialised memory, so what remains is (ii), the problem of
determining, once we are convinced that a pointer corresponds to a particular
block, that it is actually pointing within the bounds of the block.

  [^block]: We use the term *block* to refer to an *entire* contiguous region of
  memory. The Standard uses the general term *object* to refer to “a region of
  data storage”, but it doesn’t have a term that emphasises an entire contiguous
  section in distinction from any region. So every block is an object, but also
  every subset of a block is also an object. Our *block* is roughly equivalent
  to what the Standard calls a
  [space](https://port70.net/~nsz/c/c89/c89-draft.html#4.10.3), but we’re
  hesitant to use this word because the Standard never defines it, and the word
  “space” seems to cast into abeyance the boundedness of what we call blocks.

In regard to (ii) we’ve identified four successive kinds of buffer accesses, in
connection with the nature of knowledge that the function under verification has
about the size of the buffer and the indices where it is accessed:

1. [Local-constant size and index](https://github.com/xr0-org/xr0/issues/57).
   This is where the array or buffer under consideration has a size that is
   constantly defined within the function, and the accesses are similarly
   constantly defined indices:

2. [Local-constant size](https://github.com/xr0-org/xr0/issues/58). The size of
   the buffer is fixed as in (1.) but the accessed index is given as a
   parameter.

3. [Local-constant index](https://github.com/xr0-org/xr0/issues/59). The
   accessed index is fixed, but the buffer is now provided to the function as a
   parameter, meaning it must reason abstractly about its size.

4. [The general case](https://github.com/xr0-org/xr0/issues/60). Neither the
   size of the buffer nor the indices accessed are fixed; abstract reasoning
   about the buffer.

## Next up

We’re currently in the middle of (1.) above, and our aim is to finish the
protection for all the cases we can think of. So our goal for next week (15th)
is to close this case off, tracked under
[[#57](https://github.com/xr0-org/xr0/issues/57)].
