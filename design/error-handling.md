# Error handling

Newer systems languages (Rust, Zig, Hare) boast of an [algebraic] error handling
paradigm that statically ensures error values are handled. The basic idea is
that the return value of any given function can have union type (this is the
algebraic part), with the union involving the regular return type and a special
error type. Thus when the caller interacts with the returned value it must
safely consider the possibility of an error typed value being returned.

  [algebraic]: https://en.wikipedia.org/wiki/Algebraic_data_type

We want to investigate implementing this for C because it appears to be a
self-contained feature that only involves communicating information through
function interfaces (and shouldn't require like loop invariants).

## `.err`

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
We propose annotating in the following way:

```C
FILE *
fopen(const char *filename, const char *mode) ~ [
        if ([!]) {
                return [?0, 1?];
        }
        return .err(NULL);
];
```

or, more tersely:

```C
FILE *
fopen(const char *filename, const char *mode) ~ [
        return [!] ? [?0, 1?] : .err(NULL);
];
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
is no issue in calling `fgetc(f)`.
