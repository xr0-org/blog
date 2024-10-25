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

### Setup invariants

The above example gives us a good starting point. In order to safely execute the
body

```C
if (!(i < 9)) break;
i++;
```

we must first know that `i` has a value. So the invariant must state this
assumption:

```C
int i = 0;
while (1) ~ [ setup: i = [?]; ] {
        if (!(i < 9)) break;
        i++;
}
```

So the user would write something like:

```C
int i = 0;
while (i < 9) ~ [ setup: i = [?]; ] {
        i++;
}
```

Now this may seem silly in light of the fact that `i` is defined right before
the loop, but it is critical because in general

1. Previous executions of the loop body may invalidate what is true initially

2. It ensures us that the interface between the loop and the outside world is
   the invariant.

#### Example

A more striking example (in which we ask the reader to ignore the possibility of
a failed allocation) is

```C
int i;
void *p = malloc(10);
for (i = 0; i < 10; i++) {
        p[i] = malloc(1);
}
```

Translating into while(1) form we get

```C
int i;
void *p = malloc(10);
i = 0;
while (1) {
        if (!(i < 10)) break;
        p[i] = malloc(1);
        i++;
}
```

Here the invariant must insist that `p[i]` is well-defined before each
iteration (if `i < 10`):

```C
int i = 0;
void *p = malloc(10);
i = 0;
while (1) ~ [
        setup: i = [?];
        if (i < 10) setup: p = .clump(1)-i;
]{
        if (!(i < 10)) break;
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
        if (i < 10) setup: p = .clump(1)-i;
]{
        p[i] = malloc(1);
}
```

### Effect invariants

The keen reader will have observed that we didn't annotate the side effect

```C
p[i] = malloc(1);
```

above.
This would be done in the following way:

```C
int i;
void *p = malloc(10);
i = 0;
while (1) ~ [
        setup: i = [?];
        if (i < 10) {
                setup: p = .clump(1)-i;
                p[i] = malloc(1);
        }
]{
        if (!(i < 10)) break;
        p[i] = malloc(1);
        i++;
}
```

The sense of this annotation is that a given iteration of the loop will, if
`i < 10`, upon assumption of the `setup`, assign `malloc(1)` to the `i`-th
element of `p`.
It is therefore still a wholly internal consideration.

### Net effect

In order to refer to the effect of the loop as a whole, we have to extend our
syntax further.
In the first place, the invariant needs to capture the progress that previous
iterations accomplish:

```C
int i;
void *p = malloc(10);
i = 0;
while (1) ~ [
        setup: i = [0?11];
        if (i < 10) {
                setup: {
                        int j; /* dummy variable */

                        p = .clump(i+1);
                        for (j = 0; j < i; j++) p[j] = malloc(1);
                }
                p[i] = malloc(1);
        }
]{
        if (!(i < 10)) break;
        p[i] = malloc(1);
        i++;
}
```

Expanding the setup for `p` to `.clump(i+1)` simply makes space for `p[0]`,
`p[1]`, ..., `p[i]`.
The really interesting line is

```C
for (j = 0; j < i; j++) p[j] = malloc(1);
```

What is meant by this is that at setup we assume the first `i` assignings have
occurred.
Then

```C
p[i] = malloc(1)
```

accomplishes the `i+1`-st one.
This facilitates our annotating the net effect of the loop, which appears as
follows:

```C
int i;
void *p = malloc(10);
i = 0;
while (1) ~ [
        setup: {
                int j;

                i = [0?11];
                p = .clump(10);
                for (j = 0; j < i; j++) p[j] = malloc(1);
        }
        if (i < 10) {
                p[i] = malloc(1);
                i++;
        }
]{
        if (!(i < 10)) break ~ [
                /* `i' is a dummy variable below */
                for (i = 0; i < 10; i++) p[i] = malloc(1);
        ];
        p[i] = malloc(1);
        i++;
}
```

```C
int i;
void *p = malloc(1);
for (i = 0; i < 10; i++) ~ [
        setup: i = [0?11];
        if (i < 10) {
                setup: {
                        int j;

                        p = .clump(i+1);
                        for (j = 0; j < i; j++) p[j] = malloc(1);
                }
                p[i] = malloc(1);
        }
][
        /* `i' is a dummy variable below */
        for (i = 0; i < 10; i++) p[i] = malloc(1);
]{
        p[i] = malloc(1);
}
```

The net effect is the result of plugging `i = 10`, which follows directly from
the if-condition, into the invariant

```C
for (j = 0; j < i; j++) p[j] = malloc(1);
```

(and changing the dummy variable from `j` to `i`).

#### Example: partial action

But there is a problem. What happens when the context prior to the loop
accomplishes part of the alleged net effect?

```C
int i;
void *p = malloc(10);
p[0] = malloc(1);
p[1] = malloc(1);
p[2] = malloc(1);
i = 3;
while (1) ~ [
        setup: i = [0?11];
        if (i < 10) {
                setup: {
                        int j;

                        p = .clump(i+1);
                        for (j = 0; j < i; j++) p[j] = malloc(1);
                }
                p[i] = malloc(1);
        }
]{
        if (!(i < 10)) break ~ [
                for (i = 0; i < 10; i++) p[i] = malloc(1);
        ];
        p[i] = malloc(1);
        i++;
}
```

The net effect annotated above is clearly false, since only

```C
for (i = 3; i < 10; i++) p[i] = malloc(1);
```

will be accomplished. But, on the other hand, it is the direct result of
plugging `i = 10` into the invariant

```C
for (j = 0; j < i; j++) p[j] = malloc(1);
```

So how would we detect that it is false?
This confusion arises because the invariant is talking about what is true,
whereas the net effect about what the loop has accomplished.

## External verification

- objects potentially affected by loop have rconst value afterward
