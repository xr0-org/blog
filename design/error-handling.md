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

## An example

Consider what happens when calling this function:

```C
FILE *fopen(const char *filename, const char *mode);
```

The trouble is that, ordinarily, a C programmer might forget to handle this
error, writing code like

```C
FILE *f = fopen("/invalid/path/to/file", "r");
fgetc(f);
```

Running this (on my machine) unsurprisingly leads to a segfault. 
The underlying problem is a breakdown in assumptions.
`fgetc` requires a stream (i.e. a pointer to a `FILE *` representing an opened
file).
On the other hand, although `fopen` usually returns such a stream, 
the Standard [tells us](https://port70.net/~nsz/c/c89/c89-draft.html#4.9.5.3)
tells us 

> If the open operation fails, `fopen` returns a null pointer.

Thus `fopen` can return two _types_ of values: a pointer to a stream, or an
error, but `fgetc` is equipped to handle only the former type.
If we extend C's type system to express this distinction, we will regard `fopen`
as returning _either_ of these types, and only permit passing the stream kind to
`fgetc`.

## Annotating sentinels with `.err`

We propose annotating in the following way:

```C
FILE *
fopen(const char *filename, const char *mode) ~ [ if ([!]) return .err(NULL); ];
```

Thus a caller of `fopen` must consider the branch where the return value is
`.err(NULL)`.
If we regard the default or assumed meaning of a prototype that doesn't specify
readiness to deal with error values, like

```C
int
fgetc(FILE *stream);
```

to disallow for error values, we can catch an error in code like this,

```C
FILE *f = fopen("/invalid/path/to/file", "r");
fgetc(f);
```

```
`fgetc' expects a FILE * but `f' may be `.err(NULL)'
```

This can obviously be handled by checking:

```C
FILE *f = fopen("/invalid/path/to/file", "r");
if (!f) exit(EXIT_FAILURE);
fgetc(f);
```

The `!f` branch corresponds to that in which `.err(NULL)` is returned, so there
is no issue in calling `fgetc(f)` if it isn't selected.

## A harder example

As usual, the real complexity comes when we consider loops. 
It often happens that we construct an array of entities, e.g.

```C
void **
allocmany(int n)
{
        int i;

        void **arr = malloc(sizeof(void *)*n);
        if (!arr) return NULL;
        for (i = 0; i < n; i++) {
                arr[i] = malloc(1);
                /* handle error ? */
        }
        return arr;
}
```

Assuming `malloc` is annotated similarly to `fgetc`, i.e.

```C
void *
malloc(size_t size) ~ [ if ([!]) return .err(NULL); ];
```

the difficulty comes in deciding how to handle errors returned by `malloc` _in
the loop_ in `allocmany`, or how to communicate the necessity of doing so to its
callers.
We can see three basic ways of doing this, or rather, two fundamental
approaches, one sensible, the other ill-advised in most cases, and a spectrum of
infinitely risky compromises between them:

1. Handle all the errors inside the loop, leaving no burden to the caller
   (except to handle any errors directly signalled by `allocmany`)

2. Handle all the errors in the caller

3. Handle some of the errors and leave the caller to handle the rest (folly
   herself).

### 1. Handle errors inside the loop

This is the wisest option (in most cases).

```C
void **
allocmany(int n) ~ [ if ([!]) return .err(NULL); ]
{
        int i;

        void **arr = malloc(sizeof(void *)*n);
        if (!arr) return NULL;
        for (i = 0; i < n; i++) {
                arr[i] = malloc(1);
                if (!arr[i]) {
                        int j;

                        for (j = 0; j < i; j++) free(arr[j]);
                        free(arr);
                        return NULL;
                }
        }
        return arr;
}
```

### 2. Handle all the errors in the caller

This option is a lot riskier, because we must pass the obligation of checking
any element in the array before using it, but its advantage is uniformity: the 
the caller can regard all the returned values as _either_-type (i.e. either
`NULL`-error or valid pointer).

```C
void **
allocmany(int n) ~ [
        void **arr = malloc(sizeof(void *)*n);
        if (!arr) return arr;
        for (i = 0; i < n; i++) {
                if ([!]) arr[i] = .err(NULL);
        }
        return arr;
]{
        int i;

        void **arr = malloc(sizeof(void *)*n);
        if (!arr) return NULL;
        for (i = 0; i < n; i++) {
                arr[i] = malloc(1);
        }
        return arr;
}
```
