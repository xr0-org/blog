# Loop verification

The two main questions in connection with a loop (or any other construct) are

1. Does it violate any of the safety semantics we are verifying?

2. How does its execution (as a net effect) affect the safety properties
of the state?

So we want to verify that the loop itself is safe, and then track its impact of
the state. As an example of the former, if the loop body calls free on some
variable, we must be able to show that this variable always has a deallocand
value at this moment. For the latter, if the loop allocates memory in some
variable, we need to take account of this side effect.

Loop verification can be distinguished into internal and
external aspects.

## Internal verification

Basically we want to show that, given the initial conditions, it is impossible
for any leak or UB[^ub] to occur, and then that the annotated side effects will
occur.

  [^ub]: By _UB_ here we mean the instances of UB that we are currently
  verifying — use of uninitialised memory, dereferencing of pointers not
  pointing at valid blocks and invalid frees like double frees.


Like in the verification of a function, an integral part of our ability to do
verification comes down to the spec provided by the user. The spec gives us both
our initial conditions (in the `setup` for a function) and the net effect we
measure against. In the loop internal verification case this comes down to the
invariant. The invariant gives us initial conditions, because we verify that it
is initially true given the context in which the loop appears. The invariant
also gives us the net effect of the loop body (not of the loop) because the
body's task is to maintain the truth of the invariant.

### While(1) form

The essential character of a loop is repetition. So every loop can be expressed
in the form

```C
while (1) { /* do things */ }
```

This form is preferable to that with an explicit termination condition, such as

```C
while (*p) { /* do things */ }
```

because in the latter we will be led to verifying the evaluation of the
termination condition separately from the loop body. So we express this instead
as

```C
while (1) {
        if (!*p) break;
        /* do things */
}
```

We call this _while(1) form_.
It emphasises that the invariant is the truly characteristic interface to the
loop: everything important must be related to it.

#### Example

```C
int i = 0;
while (i < 9) {
        i++;
}
```

is rephrased as

```C
int i = 0;
while (1) {
        if (!(i < 9)) break;
        i++;
}
```

### Setup

The above example gives us a good starting point. In order to safely execute the
body

```C
if (!(i < 9)) break;
i++;
```

we must first know that `i` has a value.
This is the first thing to verify.

#### Example

A more striking example (in which we ask the reader to ignore the possibility of
a failed allocation) is

```C
int i;
void *p = malloc(10);
for (i = 0; i != 10; i++) {
        p[i] = malloc(1);
}
```

The reason for using `i != 10` instead of the traditional `i < 10` will become
clear presently.
Translating into while(1) form we get

```C
int i;
void *p = malloc(10);
i = 0;
while (1) {
        if (!(i != 10)) break;
        p[i] = malloc(1);
        i++;
}
```

Here the invariant must insist that `p[i]` is well-defined before each
iteration (if `i != 10`):

```C
int i = 0;
void *p = malloc(10);
i = 0;
while (1) ~ [
        setup: i = [?];
        if (i != 10) setup: p = .clump(1)-i;
]{
        if (!(i != 10)) break;
        p[i] = malloc(1);
        i++;
}
```

Returning to its original form we would expect the user to right something like

```C
int i;
void *p = malloc(10);
for (i = 0; i != 10; i++) ~ [
        setup: i = [?];
        if (i != 10) setup: p = .clump(1)-i;
]{
        p[i] = malloc(1);
}
```
