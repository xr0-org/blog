---
published: Jun 15, 2024
---

# #17: 1D bounds

## tl;dr

We have [local-constant size buffers](https://github.com/xr0-org/xr0/issues/57)
with constant index accesses working for one-dimensional arrays. We’re taking a
slight detour to implement multi-dimensional arrays.

## Details

Xr0 can now detect out-of-bounds accesses for the case of one-dimensional arrays
and pointers. Some examples of this:

### Out-of-bounds access on a pointer to a variable on the stack.

Xr0 rightly rejects `p[1] = 1;` since `p` is pointing at the memory location of
`i`. The code does not allocate memory beyond `i`, and accessing `p[1]`.

```C
void
foo()
{
    int i; int *p;

    p = &i;
    p[1] = 1;
}
```

```
foo.x:8:10: undefined indirection: out of bounds (foo)
```

### Out-of-bounds access to a heap-allocated region.

Xr0 rightly rejects the assignment `p[2] = 3;`. The heap location pointed to by
`p` has a size of 2, so the access `p[2]` is out of bounds.

```C
#include <stdlib.h>

void
bar()
{
    int *p;

    p = malloc(2);
    p[0] = 5;
    p[1] = 4;
    p[2] = 3;
}
```

```
bar.x:10:10: undefined indirection: out of bounds (bar)
```

### Out-of-bounds access to a stack-allocated array.

Xr0 rightly rejects the assignment `arr[3] = 5;` to the stack allocated array.
The array `arr` has a size 3, so the access `arr[3]` is out of bounds.

```C
void
baz()
{
    char arr[3];

    arr[2] = 7;
    arr[3] = 5;
}
```

```
bar.x:6:10: undefined indirection: out of bounds (baz)
```

## Next up

As mentioned, we’re taking a slight detour from [the buffer out-of-bounds problem](https://github.com/xr0-org/xr0/milestone/1)
to implement multi-dimensional arrays
([[#61](https://github.com/xr0-org/xr0/issues/61)]).  The idea is that this will
help us write more interesting tests for the bounds checking down the line. Our
goal is to finish this by the 22nd.
