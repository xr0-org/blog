---
author: [amisi]
---

# Loops in Xr0: a proposal

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


## Invariants

We verify functions in Xr0 using [the golden rule][golden] that the
spec is the interface between the function body and callers.
Internal verification is then verifying that the body implements the spec, and
external verification is ensuring that the caller satisfies the spec's
requirements and takes account of its side effects and return value.
We intend to follow a similar approach with loops, where the analogue to the
spec is the loop invariant.

  [golden]: https://steveklabnik.com/writing/rusts-golden-rule

A loop _invariant_ is a condition that is true at the beginning of the loop and
whose truth the execution of the body preserves.
Our work in verification can then be divided once again into internal and
external aspects:

- Internal: verifying that upon assumption of the invariant the loop body
  preserves it

- External: verifying that the invariant is initially true and deducing the net
  effect of the loop from the invariant and the invalidation of termination
  conditions.


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
It emphasises that the invariant is the truly characteristic interface to the
loop: everything important must be related to it.
Even the termination condition is simply a part of the loops body that
communicates with the outside world through the invariant.

## Example

Consider `foo` beneath:

```C
foo(const unsigned int len)
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

We annotate it in the following way:[^while-1]

```C
foo(const unsigned int len)
{
        int i; void **arr;

        arr = malloc(sizeof(void *)*len);
        assert(arr);

        for (i = 0; i < len; i++) ~ [
                int j;
                i = [0?len+1];
                for (j = 0; j < i; j++)
                    arr[j] = .malloc(1);
        ]{
                arr[i] = malloc(1);
                assert(arr[i]);
        }

        for (i = 0; i < len; i++) ~ [
                int j;
                i = [0?len+1];
                for (j = 0; j < i; j++)
                    free(arr[j]);
        ]{
                free(arr[i]);
        }

        free(arr);
}
```

  [^while-1]: It is important to emphasise that the invariants apply to the
  while(1) form of the loops. Taking the first loop as an example:

    ```C
    i = 0;
    while (1) ~ [
    	int j;
    	i = [0?len+1];
    	for (j = 0; j < i; j++)
                arr[j] = .malloc(1);
    ]{
    	if (!(i < len)) break;
    	arr[i] = malloc(1);
    	assert(arr[i]);
    	i++;
    }
    ```

### Algorithm

1. _Derive invariant state._ Taking the state as it is in context (e.g. with
   `arr` holding a pointer to an allocated region for the first loop) execute
   the invariant.
   The state now present will be referred to as the _invariant state_.

2. _Setup_. Verify that the context satisfies the invariant state, using the
   same techniques as in setup verification for functions.

3. _Execute_ the loop body on the invariant state. Use looping branches
   for _invariance_ verification and terminating branches for computing the
   _net effect_ of the loop:

    - _Invariance_. Verify the final state state in any branch that gets to the
      end of the loop body for satisfaction of the invariant state, just as in
      (2).

    - _Net effect_. Whenever a loop-terminating statement (`break`, etc.) is
      encountered, regard the state as representative of the net effect of
      the loop, since it is the concatenation of the invariant state and the
      statements between the beginning of the loop body and the termination
      point. In the first loop above, we should get the following as the net
      effect:

        ```C
        i = len;
        for (j = 0; j < len; j++)
                arr[j] = .malloc(1);
        ```

        Both statements are just the invariant with `i >= len`.
