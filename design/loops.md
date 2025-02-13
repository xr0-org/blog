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

1. Internal: verifying that the loop is well-defined

2. External: taking account of its effect upon the state.

(1.) reduces to verifying that all the possible executions of the loop body,
given the initial conditions, are well defined.
In other words, we must establish _invariant well-definedness_.
This involves identifying the necessary preconditions of well-defined execution
of the body of the loop, i.e. its setup, and then showing

- That the setup is initially satisfied (base step)

- That upon assumption of the setup, an execution of the loop body will
  re-establish the same setup (inductive step).

We can and should investigate deducing setup (and other parts of specifications)
from bodies, but that is work for another day. So we will begin by assuming that
the user provides the setup for a loop. If this setup happens to be more
restrictive than is necessary, there is no danger, but if it is less then Xr0
must detect this.

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

Let us begin with internal verification of the first loop: 

```C
for (i = 0; i < len; i++) ~ [
	setup: i = [0?len+1];
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
	setup: i = [0?len+1];
	if (i < len) setup: arr = .clump(sizeof(void *))-i;
]{
	if (!(i < len)) break;
	arr[i] = malloc(1);
	assert(arr[i]);
	i++;
}
```

This seems awfully complicated. The culprit is the termination condition
	`i < len`,
which is underspecific.
If we change to `i != len`, we get a simpler invariant:

```C
i = 0;
while (1) ~ [ if (i != len) setup: arr = .clump(sizeof(void *))-i; ] {
	if (!(i != len)) break;
	arr[i] = malloc(1);
	assert(arr[i]);
	i++;
}
```

or

```C
for (i = 0; i != len; i++) ~ [
	if (i != len) setup: arr = .clump(sizeof(void *))-i;
]{
	arr[i] = malloc(1);
	assert(arr[i]);
}
```

The second loop (adjusted to `i != len`) must require both the space to
dereference and the prior allocation:

```C
for (i = 0; i != len; i++) ~ [
	if (i != len) setup: {
		arr = .clump(sizeof(void *))-i;
		arr[i] = malloc(1);
	}
]	free(arr[i]);
```

Putting things together, we get

```C
foo(unsigned int len)
{
	int i; void **arr;

	arr = malloc(sizeof(void *)*len);
	assert(arr);

	for (i = 0; i != len; i++) ~ [
		if (i != len) setup: arr = .clump(sizeof(void *))-i;
	]{
		arr[i] = malloc(1);
		assert(arr[i]);
	}

	for (i = 0; i != len; i++) ~ [
		if (i != len) setup: {
			arr = .clump(sizeof(void *))-i;
			arr[i] = malloc(1);
		}
	]	free(arr[i]);

	free(arr);
}
```

