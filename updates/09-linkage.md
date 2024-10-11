---
published: Jan 12, 2024
author: [team]
---

# #9: Linkage

_We're experimenting with the style of our updates, trying to make them more
engaging. Let us know what you find interesting (or boring!) as we continue._

## tl;dr

We implemented the first version of internal linkage of functions, allowing them
to be forward declared in prototypes. For our next iteration, we will be focused
on verifying the rest of [parse.x].

  [parse.x]: https://git.sr.ht/~lbnz/xr0/tree/d3bb4a4cd3f8e989c6da9260c41e32aaf7a467e6/item/tests/3-program/100-lex/parse.x

## Introduction

In order to make Xr0 usable on practical programs, our aim for the last few
weeks has been to verify [parse.x], a simple parser for the [Lex] input format.
Recently we managed to verify all the functions of `malloc` and `free` (with the
exception of main), and in the [last iteration] we added `*.c` file code
generation (from `*.x` files) to Xr0. The next challenge is to get the remaining
functions in the parser verified, but first we needed the ability to forward
declare functions, which is what we added in this iteration.

Forward declaration of functions in C works through what is called _linkage_,
and we've spent the last few days implementing a simplified form of linking for
functions within individual _translation units_ (defined beneath).

In our update today we will begin by summarising how linkage works in C and then
explain the form in which we've implemented this for Xr0.

(Unless specified otherwise, _linkage_ below refers to _linkage of functions_.)

  [Lex]: https://en.wikipedia.org/wiki/Lex_(software)
  [last iteration]: https://open.substack.com/pub/xr0blog/p/61a?r=38hkcl

## Linkage in C

Linkage is the way identifiers can be made to transcend particular _scopes_ in
C. In particular, it's the process by which _function prototypes_ are associated
to _function definitions_, i.e. how the `a`'s below are tied together:

```
a();

b() { a(); }

a() { /* code */ }
```

Linkage is defined in relation to two important concepts in C, that of
_translation unit_ and _scope_. C is designed so that programs can be compiled
piece-by-piece rather than all at once; this is helpful on large programs and
when making use of shared libraries. It was also of particular importance when
machines were slower than today. C compilers, therefore, technically only
ever[^ever] compile one file at a time, and a source file together with all the
headers and other source files included via preprocessor `#include`s is called a
_translation unit_.

An identifier's _scope_ is the region of program text within which it is visible.
There are [four kinds of scopes]: _function_, _file_, _block_ and _function
prototype_. Of these, file scope is the most relevant to linking function
prototypes with function definitions. File scope applies to any identifier not
declared inside a function or a list of parameters, and begins from the moment
of declaration to the end of the translation unit. Above, the function `b`
cannot "see" `a`'s definition: it is out of scope. It can only see the prototype
that declares `a` above it. The process by which these two declarations – the
prototype that "forward declares" `a` and the function definition – are
understood by the compiler to refer to the selfsame function is what is meant by
_linkage_.

  [four kinds of scopes]: https://port70.net/~nsz/c/c89/c89-draft.html#3.1.2.1

The Standard defines [three kinds of linkage], _external_, _internal_ and
_none_. If an identifier has external linkage it refers to the same object or
function no matter where it is in the whole program. If it has internal linkage
it refers to the same object or function within its translation unit. Finally,
if it has no linkage it refers to a unique object that is inaccessible outside
its scope.

  [three kinds of linkage]: https://port70.net/~nsz/c/c89/c89-draft.html#3.1.2.2

Functions always have linkage: external linkage by default or also if they're
declared with the storage-class `extern`, and internal linkage if declared with
static.

## How we've added linkage to Xr0

The unique challenge for Xr0 here is we have to choose rules for determining the
abstract[^abstract] of a function whenever it is invoked, because this is the
only way to simulate its effect in the caller's state. These rules must also
govern where the abstracts may or must appear, in prototypes and definitions. As
always we desire that Xr0 be a kind of "analytic continuation" of C, the organic
extension of Ritchie's design philosophy.

For the moment, since we're focusing on linkage within a translation unit alone,
we've implemented linkage according to these rules:

1. The first appearance of a function must be accompanied by an abstract

2. All abstracts that appear for a function must be equivalent[^eq]

3. Function prototypes must have abstracts.

Rule (1.) is justified by the [Rationale for C89][rat], which explains that

> The Standard requires that the first declaration, implicit or explicit, of an
> identifier specify (by the presence or absence of the keyword `static`) whether
> the identifier has internal or external linkage. This requirement allows for
> one-pass compilation in an implementation which must treat internal linkage
> items differently than external linkage items.

  [rat]: https://port70.net/~nsz/c/c89/

Thinking along these lines, we want Xr0 to be able to discern a function's
abstract from its first appearance, so that in a single pass it can simulate the
function's execution wherever it is invoked.

Rule (2.) is simply that of well-defined-ness.

Finally, rule (3.) is the most questionable because it requires the user to copy
around the abstract whenever repeating a prototype. However, since there is no
utility in copying around a prototype (instead of moving it higher in the source
file) it has no practical impact for using Xr0 in its current form. We will
likely revise it when we revisit linkage in the future. We leave it as it is
because our implementation involves this restriction and we don't want to invest
more time reflecting on linkage before reaching our milestone.

## Next up

The milestone is, of course, verifying [parse.x]. Our goal for next few days (to
20th) is to get the whole file parsed. While this is perhaps slightly ambitious,
it allows us to see if there are any fundamental roadblocks left in between
where we are and this achievement.

Goal:

1. Verify parse.x while it is working end-to-end
   [[#21](https://todo.sr.ht/~lbnz/xr0/21)].


Stretch goals:

1. Simplify map and array data structures
   [[#13](https://todo.sr.ht/~lbnz/xr0/13)]

2. Write a first-version of the "topological sort" mechanism
   [[#16](https://todo.sr.ht/~lbnz/xr0/16)].



  [^ever]: When you run

  ```
  cc a.c b.c
  ```

  the compiler actually produces two object files and then links them together,
  treating the two files as separate translation units. 

  [^abstract]: An abstract in Xr0 is the `~[]`-delimited region that appears
  after a function's signature and captures its semantic effect on the state.
  See here for more. 

  [^eq]: We're using equality of the string forms as a proxy for semantic
  equivalence later, which would permit, for example, variation in
  (abstract-)local variable names. 
