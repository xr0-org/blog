---
author: [amisi]
---

# Loops

In approaching the problem of loops, our focus must first and foremost be in
identifying a useful subspace within C that is relatively easy to verify.
Our commitment to ensuring that all well-defined C is expressible within Xr0
must not distract us.

Our [Lex parser][parse.x] furnishes the right starting point.
The issues involved in communicating the semantics of operations like

```C
for (i = 0; i < n; i++) {
	parsed = parse_token(pos);
	t[i] = *parsed.tk;
	free(parsed.tk);
	pos = skipws(parsed.pos);
}
```

and

```C
for (i = 0; i < arr->n; i++) {
	free(arr->t[i].name);
	free(arr->t[i].action);
}
```

are sufficiently profound to occupy our attention first.

  [parse.x]: https://github.com/xr0-org/xr0/blob/master/tests/0v/99-program/100-lex/parse.x

## Internal and external verification

Like any other construct, in verifying a loop we have two concerns:

1. Internal: verifying that the loop is a well-defined operation

2. External: taking account of its effect upon the state.

(1.) reduces to verifying that all the possible executions of the loop body,
given a set of clearly identified assumptions, are well defined.
In other words, we must establish _invariant well-definedness (IWD)_.
This involves identifying the necessary preconditions of well-defined execution
of the body of the loop, i.e. its setup, and then showing

- That upon assumption of the setup, execution of the loop body does not lead to
  UB (well definedness)

- That upon assumption of the setup, execution of the loop body
  produces a state that satisfies the same setup (inductive step of the IWD
  proof).

We can and should investigate deducing setup (and other parts of specifications)
from bodies, but that is work for another day. So we will begin by assuming that
the user provides the setup for a loop. If this setup happens to be more
restrictive than is necessary, there is no danger, but if it is less then Xr0
must detect this.

(2.) also has two aspects:

- Verifying that the setup of the loop is satisfied in the place it appears in
  the function (base case of the IWD proof)

- Taking account of the net effect of the loop upon the state.

In reality (1.) and (2.) are not entirely separable from one another, because to
establish the net effect of the loop, one has to first do internal verification
of the same, augmenting the invariant with the relevant side effects or insights.

## While(1) form

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

For example:

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

We will regard all other forms of loops as abbreviations for while(1)-form
loops.

## Example

Idealising somewhat, consider `foo` beneath:

```C
foo(unsigned int len)
{
	int i; void **arr;

	arr = malloc(sizeof(void *)*len);
	assert(arr);

	for (i = 0; i < len; i++) {
		arr[i] = malloc(1);
		assert(arr[i]);
	}

	for (i = 0; i < len; i++)
		free(arr[i]);

	free(arr);
}
```

### Internal verification

Let us begin with the first loop:[^init]

```C
for (i = 0; i < len; i++) ~ [
	if (i < len) setup: arr = .clump(sizeof(void *))-i;
]{
	arr[i] = malloc(1);
	assert(arr[i]);
}
```

It is important to stress that the annotation applies to the beginning of the
while(1)-form version of the loop:

```C
i = 0;
while (1) ~ [
	if (i < len) setup: arr = .clump(sizeof(void *))-i;
]{
	if (!(i < len)) break;
	arr[i] = malloc(1);
	assert(arr[i]);
	i++;
}
```

  [^init]: It is worth asking whether we need to annotate the assumption that
    `i` and `len` are defined:

	```C
	i = 0;
	while (1) ~ [
		setup: { i = [?]; len = [?]; }
		if (i < len) setup: arr = .clump(sizeof(void *))-i;
	]{
		if (!(i < len)) break;
		arr[i] = malloc(1);
		assert(arr[i]);
		i++;
	}
	```

    Presently we don't annotate such an assumption for function parameters,
    because the call expression that leads to a function's execution must
    provide values for all the parameters. This is not the case with loops,
    which can be placed at a point where some local variable is uninitialised.
    My suggestion for now is we operate under the rule that every variable used
    without initialisation in a loop is _implicitly_ understood to have been
    initialised, so that the setup form above is permissible, but not redundant. 


The second loop must require both the space to
dereference and the prior allocation:

```C
for (i = 0; i < len; i++) ~ [
	if (i < len) setup: {
		arr = .clump(sizeof(void *))-i;
		arr[i] = malloc(1);
		assert(arr[i]);
	}
]	free(arr[i]);
```

Internal verification of each of these loops is straightforward now.
That of the first loop, for example, is comparable to verifying the following
function (ignoring side effects):

```C
bar(int i, unsigned int len, void **arr) ~ [
	if (i < len) setup: arr = .clump(sizeof(void *))-i;
]{
	if (!(i < len)) return;
	arr[i] = malloc(1);
	assert(arr[i]);
	i++;
}
```

The second loop would be similar. The key difference, in each of these cases,
is we have to verify that at each of the return points (the `return` in the
`if`-condition and after `i++;` in `bar` above) that the setup is satisfied.

### External verification

External verification of the first loop is pretty straightforward, it is exactly
the same thing as setup-verifying a call to `bar` right before the while(1) form
of the loop, i.e. after `i = 0;` is executed.

```C
foo(unsigned int len)
{
	int i; void **arr;

	arr = malloc(sizeof(void *)*len);
	assert(arr);

	i = 0;
	bar(i, len, arr);

	/* ... */
}
```

This would obviously work, because if `i < len`, since `i == 0`, it follows that
`len > 0`, so that the allocation for `arr` would have a nonempty region,
establishing `arr+i == arr+0 == .clump(sizeof(void *))`.

The second loop, however, poses some challenges, because its setup requires
successful allocation for `arr[i]`, which occurs in the first loop (if it
terminates).
So we have to first explain how this information is captured in the state,
before proceeding to verify the second loop.
