# Algebraic error handling

Newer systems languages (Rust, Zig, Hare) boast of an [algebraic] error handling
paradigm that statically ensures error values are handled. The basic idea is
that the return value of any given function can have union type (this is the
algebraic part), with the union involving the regular return type and a special
error type. Thus when the caller interacts with the returned value it must
safely consider the possibility of an error typed value being returned.

  [algebraic]: https://en.wikipedia.org/wiki/Algebraic_data_type

We want to investigate implementing this for C because it appears to be a
self-contained feature that only involves communicating information through
function interfaces (and won't require like loop invariants for the most part).

## Benefits of algebraic error handling

(I asked ChatGPT to list these; this is my condensation of the response.)

### 1. Prevent sentinels from being treated as valid data {#no-1}

For example, `fopen` returns `NULL` if the provided file cannot be accessed:
failure to handle this can lead to undefined behaviour by calling, say,
`fgetc(NULL)`.

### 2. Elegantly propagate errors up the call stack {#no-2}

These newer languages have special syntaxes for calling a function that may
return an error and propagating the error up the call stack if encountered.
This leads to very elegant expression of ideas, because one can choose where to
handle the errors without writing a lot of boilerplate.

### 3. Provide safe error values for multithreading {#no-3}

`errno` cannot be relied upon in a multithreaded environment.

### 4. Provide richer error values {#no-4}

Return not only an indication of whether/not an error occurred, but a custom
value that may have additional information (like a string describing the error).

### Analysis

[(3)](#no-3) is inapplicable to C89 as such, so we will exclude it from our
analysis for now.

[(2)](#no-2) and [(4)](#no-4) are nice-to-haves but aren't safety
considerations at all. They are province of new language designers, and could
never be brought to C by Xr0, because our number one rule is that (un-annotated)
source remains vanilla C.

So it appears that the only benefit provably algebraic error handling can bring
to C is [(1)](#no-1).


## Generalising

Preventing sentinels from being treated as regular data is a special case of the
fundamental communication problem that Xr0 addresses.
The general situation is there is a safety requirement at one point in the
codebase whose satisfaction depends upon information visible at another.
Is this pointer dereferencable? Can we call `free` on this pointer? Can we treat
this `FILE *` as a stream and pass it to `fgetc`?
In every case it comes down to rigorous communication from the point where an
object is defined to the point where it is used.

The advantage this error-handling problem may provide is it is a more flexible
and narrow amount of information that we need to communicate. Verifying memory
safety requires many things (valid pointer, not freed or referring to a popped
stack frame, within bounds, and referring to initialised memory — if we're
reading its referent).
A more ideal candidate for production ready use may be communicating
user-supplied assertions.

For example:

```C
#define FOO_INT "foo_int"

axiom int
foo_make() ~ [ return .tag(FOO_INT, [?]); ];

axiom
foo_use(int k) ~ [ setup: k = .tag(FOO_INT, [?]); ];
```

Assuming there are no other axioms involving `FOO_INT`, any function calling
`foo_use` would then need to supply a value that was originally created by
`foo_make`. We could add to this either a keyword `.untagged` that restricts the
presence of a tag, like

```C
axiom
bar(int k) ~ [ setup: k = .untag(FOO_INT, [?]); ];
```

or, perhaps preferably, we could make it such that tags must always be stated,
so that the absence of a requirement as in `foo_use` above implies restricting
the presence thereof.

A similar proposal would be custom types, as it were:

```C
~ [ .tag .foo_int(int); ] /* defining a `.foo_int' as an integer type */

axiom int
foo_make() ~ [ return .foo_int([?]); ];

axiom
foo_use(int k) ~ [ setup: k = .foo_int([?]); ];
```

Communicating of tags of this kind through function interfaces is trivial, so
the only question is how to handle loops.


## Loops

Our basic aim here is to identify a pattern that doesn't require user-supplied
invariants for a significant subset of the kinds of loops that programmers
write. For example, if we're initialising an array of `.foo_int`-tagged
integers, as in

```C
int **
baz(int n)
{
        int i;

        int *arr = malloc(sizeof(int *) * n);
        assert(arr);
        for (i = 0; i < n; i++)
                arr[i] = foo_make();
        return arr;
}
```

we should be able to annotate and verify `baz` in a "simple" way, such as

```C
int **
baz(int n) ~ [
        int i;

        int *arr = malloc(sizeof(int *) * n);
        assert(arr);
        for (i = 0; i < n; i++)
                arr[i] = .foo_int([?]);
        return arr;
];
```

(Obviously, we should be able to use `foo_make` instead of `.foo_int` in the
annotation above with equivalent effect.)
