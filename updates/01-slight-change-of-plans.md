---
published: Nov 17, 2023
author: [team]
---

# #1: "Slight" change of plans

Having glanced at [these] reflections by the creator of [Inko] we no longer feel
convinced that focusing on self-hosting is a good idea.

  [these]: https://yorickpeterse.com/articles/a-decade-of-developing-a-programming-language/
  [Inko]: https://inko-lang.org/

We had thought that Xr0 is a special case in being annotative, with the vast
majority of our effort focused on the front-end (since the back-end’s sole job
is to strip the annotations). This made self-hosting seem practical and
achievable.

However, upon further reflection, we now see that self-hosting would require
that we maintain a "bootstrapping back-end" to strip the Xr0-annotations from
our `*.x` files, which seems like potentially-fatal overhead to accept, and
certainly one that will slow us down.

For the rest of this iteration (ends 2 Dec), we've decided to focus on a much
smaller program that deals with matrices. Our goals for it are as follows:

- It will parse (100%)

- It will verify (with the exception of loops).
