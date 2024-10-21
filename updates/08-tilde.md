---
published: Jan 08, 2024
author: [team]
---

# #8: ~

We managed to get the C code generation working.

## Why we managed

The key change that enabled us to succeed is a modification to our abstract
syntax. Previously we would write

```C
void *
foo() [ .alloc result; ];
```

in which the `[]`-delimited block contains `foo`'s abstract. The challenge with
square-bracket delimitation is `[]` is already used in C (for arrays), so the
reliable way, as far as we can tell, to remove the Xr0 annotations from a `.x`
file is to parse it with a modified C grammar.

Parsing poses further challenges because one has to handle preprocessor
directives and, in particular, `#include`'s of standard library files. It is (we
think) out of the question for us to implement a preprocessor, so our thinking
in the verification phase of Xr0 has been to rely on the preprocessor available
in the environment via `cc -E`. This works well because we're writing the
(axiomatic) headers for the standard library, because during verification the
source files are only used for axiomatic reasoning with respect to the rules in
the Standard, so we don't need a "compile-able" implementation of libc.

When doing generation, however, the code has to be compiled, so the headers
included, particularly for the standard library, must correspond to the
implementations that will be linked together into the final binary. This means
that if cc in the user's environment corresponds to GCC, then we must use GCC's
standard library implementation (or one that can be compiled with GCC), and so
on for Clang and any other compiler that Xr0 may be relying upon. But alas,
these standard libraries make use of compiler built-ins that are not
syntactically valid C, e.g. attribute syntax like

```C
char *__attribute__((aligned(8))) *f;
```

![Are you kidding me?](/updates/underwood.gif)

So in order to implement code generation in this way we would have to support
all the built-in syntactical eccentricities of the major C implementations out
there, which would impose a permanent and enormous compatibility burden for the
Xr0 project. (Perhaps there's a clever way to do this without changing the
syntax, but (a) we haven't thought of it and (b) do we want to be that clever?)

We settled, therefore, after the thoughtful deliberation of many hours (by which
we mean the vain attempt to implement generation in the above way) on adding a
disambiguating tilde before the square-brackets for our annotations:

```C
void *
foo() ~ [ .alloc result; ];
```

This helps because nowhere in valid C does the sequence `~[` or `~ [` occur, so
we can use very simple find-and-replace logic to strip out the Xr0 annotations.

We also like it because `~` is often used to signify equivalence, which brings
out more explicitly how we think of abstracts.

## Next up

Although our implementation of the generation phase of Xr0 is very rough, it's
good enough to allow us to generate compile-able code. We'll spend the rest of
this week (to 13th) figuring a workable solution for linking function prototypes
with definitions (within a file) so we can get parse.x to compile.

Goals:

1. Write tests for internal linkage of functions that correspond to parse.x
   [[#20](https://todo.sr.ht/~lbnz/xr0/20)]

2. Implement low-res solution to get these to verify
   [[#21](https://todo.sr.ht/~lbnz/xr0/21)]

3. Get parse.x to compile and execute without losing the zero-order verifications
   [[#22](https://todo.sr.ht/~lbnz/xr0/22)].

Stretch goals:

1. Simplify map and array data structures
   [[#13](https://todo.sr.ht/~lbnz/xr0/13)]

2. Write a first-version of the "topological sort" mechanism
   [[#16](https://todo.sr.ht/~lbnz/xr0/16)].
