---
published: Mar 29, 2024
author: [team]
---

# #12: Solved use-after-frees

## tl;dr

We’ve been slacking on the updates, but we’ve extended Xr0’s functionality to
reject use of uninitialised memory and use-after-free bugs.

## Introduction

Uninitialised memory access bugs and use-after-free memory bugs both fall under
the broader classification of undefined memory access bugs. Xr0 protects against
these classes of bug by rejecting instances in the program text where a value is
dereferenced in a way that leads to undefined behaviour.

The high level functionality involved in getting this to work was the
implementation of

1. “pass by pointer” functionality for the arguments to functions.

2. Encoding well defined memory access semantics which
    - Reject cases of uninitialised memory accesses
    - Reject cases of invalid pointer dereferences.

## Pass by pointer in C

Pass by pointer in C refers to a mechanism that allows one to pass a memory
address as an argument to a function rather than the actual value thereby
allowing a function to directly manipulate the data stored at that address.

```C
void
modify(int *q)
{
    *q = 1;
}

int
main()
{
    int p;

    p = 0;
    /* p == 0 */
    modify(&p); /* pass the address of p to the function modify */
    /* p == 1 */
}
```

As be seen in the `main` function above, instead of passing the value of `p` to
`modify` we instead pass a memory address of `p` via the use of the `&` "address
of" operator. Within `modify` we are then able to dereference the pointer to
access or modify the data being pointed at. In this case we assign `1` to it.

### How we implemented this in Xr0

There are two parts to this implementation. The first is the implementing the
ability to define pointers to variables declared in different stack frames, the
second was adding the `&` "address of" unary operator to the syntax and having it
semantically evaluate to return a pointer the variable with an appropriate
address and associated frame.

As a result the example above can now be expressed in Xr0 statically.

```C
void
modify(int *p) ~ [
    *p = 1;
] {
    *p = 1;
}

int
main()
{
    int p;

    p = 0;
    ~ [ p == 0 ] /* statically verify p == 0 */
    modify(&p);
    ~ [ p == 1 ] /* statically verify p == 1 */
}
```

Note how we pass the address of `p` into the `modify` function and set its value
to `1` , our verification blocks verify that the value of `p` was changed as
expected.

## Undefined memory access

Memory accesses occur during the evaluation of expressions in C. This could be
variable access, pointer dereferencing, array access, structure access to name a
few.

Undefined memory access bugs occur when a program attempts to read from or write
to a memory location that has not been properly initialised, or that has already
been freed. A discussion of undefined memory access must be underpinned by a
understanding of

### Lvalue and Rvalue semantics

There are semantically two ways in which memory is accessed; lvalue and rvalue
cases. In lvalue cases a location in memory is written (assigned) to. In rvalue
cases its value is read. Xr0 verifies that

- For lvalues, the memory location is defined (there need not be a value).

- For rvalues, the memory is defined and has a value, for example you cannot
  assign a variable that has not been initialised to another variable since it
  has no value.

### Rejecting rvalue access of uninitialised memory

Uninitialised memory bugs occur when a variable is used (as an rvalue) before it
is initialised with a proper value. In C when you declare a variable it contains
whatever value was previously stored in that memory location, if you try to use
that value without having explicitly assigning a value to it, you use whatever
"garbage" value happened to be in that location, which leads to undefined
behaviour.

#### How we implemented this in Xr0

C programs are made up of statements and expressions. Statements are
instructions that control the flow of the program; for example, branching logic
in `if` statements, or looping in `for` statements. Expressions are combinations
of operands and operators that evaluate to a single value, for example they can
be computations, assignments, comparisons etc.

Xr0 statically executes all statements within a function. The execution of these
statements recursively evaluates the statements and expressions (if any) they
may comprise of.

Given this, the task of checking for uninitialised memory access simply involves
throwing an error if an expression that is accessed as an rvalue and has no
corresponding value in our state.

Some outputs Xr0 produces in this scenario:

1. Assigning an uninitialised variable.

```C
void
undef1()
{
	int x;
	int y;
	y = x;
}
```

```
$ ERROR: undefined memory access: `x' has no value
```

2. Passing an uninitialised variable as a parameter to a function

```C
int *
undef2()
{
    int i;
    printf("x: %d\\n", i);
}
```

```
$ ERROR: undefined memory access: `i' has no value
```

### Rejecting invalid pointer dereferences

Our definitions of rvalue and lvalue semantics work in the cases that the
variables (pointers) are declared within the scope of the function under
verification, but things get interesting when we are talking about functions
that take parameters. Take for example:

```C
void
modify(int *i)
{
	*i = 1;
}

int
main()
{
	int *p;
	modify(p);
}
```

The function `modify` takes an `int *i` as a parameter and sets its value to
`1`. Some questions arise.

1. From the frame of reference of verifying the function `modify` internally, we
   need a way to specifying our assumptions about the value of parameter `*i`.

2. From the frame of reference of a caller of the function like `main`, we need
   to verify that whatever setup preconditions the function specifies/assumes
   about its parameters are met by the state of the program prior to invoking
   the function.

The assumptions of interest to us currently are; whether a parameter (that is a
pointer) is lvalue dereference-able, rvalue dereference-able or both, as well as
whether it is heap allocated.

#### How we implemented this in Xr0

From working on leaks, we already had the `.alloc` keyword that specified a
parameter (pointer) is heap allocated. In order to specify that a parameter is
dereference-able (remember we can pass by reference stack allocated variables
now) we introduced the `.clump` keyword. To illustrate:

```C
int
modify(int *i) ~ [
    setup: .clump i;
	*i = 1;
] {
	*i = 1;
}

void
caller1()
{
	int p;
	modify(&p); /* pass lvalue dererenceable pointer*/
}

void
caller2()
{
	int p;
	p = 1
	modify(&p); /* pass lvalue and rvalue dereferencable pointer */
}

void
caller3()
{
	int *p;
	p = malloc(sizeof(int));
	modify(p); /* pass heap allocated lval dereferencable pointer */
}
```

All three `caller` functions are valid given the implementation of `modify`. The
`.clump` keyword lets us express that the parameter `int *i` is lvalue
dereference-able.

If we had a `.alloc` instead of `.clump` in `modify`’s `setup` instead, then
only `caller3` would be appropriate since `p` is heap allocated. Xr0 would
reject `caller1` and `caller2`.
