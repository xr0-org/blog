---
published: Jun 22, 2024
author: [team]
---

# #18: Multi-dimensional arrays (and bounds)

## tl;dr

This week we implemented [multi-dimensional
arrays](https://github.com/xr0-org/xr0/issues/61), as well as [converged our
internal representations for declarations and
statements](https://github.com/xr0-org/xr0/issues/63).  We will be proceeding
with the buffer out-of-bounds problem for the next few weeks, beginning with the
case we call [local-constant size](https://github.com/xr0-org/xr0/issues/58),
which refers to an array whose size is fixed within the function under
verification, accessed at arbitrary indexes whose values can depend on
parameters.

## Some details

### Multi-dimensional arrays

In our work last week on the [local-constant size and
index](https://github.com/xr0-org/xr0/issues/57) case of buffer out-of-bounds
protection, we concluded that it would be convenient to implement array types,
to help us write a diverse set of tests. This turned out to be
harder than it first appeared, on account of the subtlety of C’s pointer/array
equivocation. Last week we were only able to get unidimensional arrays working,
and we’re pleased to report that the multidimensional ones are now working.

> Two ideas are most characteristic of C among languages of its class: the
> relationship between arrays and pointers, and the way in which
> declaration syntax mimics expression syntax. They are also among its
> most frequently criticised features, and often serve as stumbling blocks
> to the beginner.
> 
> Ritchie, The Development of the C Language.[^ritchie]

  [^ritchie]: With American spellings fixed.

Getting the array semantics right has been a shockingly difficult challenge, but
we find encouragement in Ritchie’s words above. The pointer/array relationship
in C is in a very real sense its essential and distinguishing characteristic,
together with its mimicking declaration syntax. To introduce arrays we’ve had to
understand both of these, and they both require a great deal of recursion.

Consider the code below as an example:

```C
char a[4][3];
a[3][2] = 'a';
```

When processing the assignment, the compiler treats `a[3][2]` as

```C
*(*(a+3)+2)
```

So evaluation of the left-hand-side begins with a the interior most sum `a+3`,
which must be recognised as the sum of a pointer and an integral offset. But `a`
has an array type, so before it can be thus regarded, a conversion must take
place in which it becomes a pointer to its first element, which is in turn an
array (of three characters). The offset must be scaled by the size of the object
pointed at, so `a+3` is internally regarded as nine addresses above the
beginning of `a`. The first indirection, `*(a+3)`, then occurs, resulting in an
array which must in turn be converted into a pointer, before being added to a
trivially-scaled[^trivially] offset, `2`, before the final indirection takes place.

  [^trivially]: Trivial because the size of the underlying elements is now 1,
  because they’re characters.

The same kind of recursive reasoning takes place also in parsing the declaration

```
char a[4][3];
```

but we will not go into the details. Suffice to say that beginners are not wrong
when they find C’s pointer/array semantics difficult. Let them take comfort
(with us) in Ritchie’s words (from the same document referenced above, our
emphasis):

> In spite of its difficulties, I believe that the C's approach to
> declarations remains plausible, and am comfortable with it; it is a
> useful unifying principle.
> 
> The other characteristic feature of C, its treatment of arrays, is
> more suspect on practical grounds, though it also has real virtues.
> Although the relationship between pointers and arrays is unusual, it
> can be learned.

### Converging declarations and statements

This is a chore we’ve been intending to do for a while. In view of the similarity
between declarations and statements (both are semicolon terminated, are
“executed”) and also the fact that we need to implement declaration
initialisers, it seemed prudent to use a single representation for them.

## Next up

The work on Xr0 continues. As mentioned above, our attention is returning to the
buffer out-of-bounds problem, and in particular, the case where the array size
is fixed by constants given within the function being verified
[[#58](https://github.com/xr0-org/xr0/issues/58)]. Our goal for the next two
weeks is to solve this case, with the possible exclusion of the general loop
cases. See you on 6th July (or earlier).
