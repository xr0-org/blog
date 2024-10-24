---
published: Apr 24, 2024
author: [team]
---

# #13: We're making Xr0 into a static debugger

## tl;dr

The work on Xr0 continues, though the updates have been consistent. We’ve spent
the past few weeks extending Xr0 with a feature we call *static debugging*,
which will allow traversing through the code in a manner reminiscent of GDB. Our
goal is to release the debugger by the end of next week (4th May).

## Introducing 0db, the static debugger

Xr0 eliminates undefined behaviour by simulating the reasoning process that
programmers do, albeit without error. In order to do this it has to compute what
is knowable on each line of code. For example, if it sees

```C
int i; int *p;

p = &i;
*p = 5;
/* etc. */
```

it has to know that initially `i` and `p` have indeterminate values[^indeterminate] that after the
first assignment p is pointing at i, that after the second i has a value of 5,
and so on.

  [^indeterminate]: If an object that has automatic storage duration is not
  initialized explicitly, its value is indeterminate. ([3.5.7 of the c89 standard](https://port70.net/~nsz/c/c89/c89-draft.html#3.5.7))

Xr0 does this by building a state which it updates as it simulates execution of
the lines, but it never pauses (unless it detects some kind of error). We’ve
been thinking for a while that it would be interesting and fun, not to mention
useful, to add
[REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) mode
in which the programmer can traverse line-by-line, GDB-style, and interrogate
the state at any point. So that’s what we’ve been adding to Xr0: we call it *0db,
the Static Debugger*. 0db differs from GDB in that it allows you to analyse your
code in terms of all the possibilities, rather than in terms of one given
situation. In other words, 0db lets you debug your code *sub specie aeternitatis*.

That’s all we can say for now — we will explain this in more detail once 0db is
running more smoothly.

## Next up

As we mentioned, our goal [[#45](https://github.com/xr0-org/xr0/issues/45)] is
to release the debugger next Saturday.

PS: If you can’t wait that long, you can play with the feature branch
[here](https://github.com/xr0-org/xr0/tree/feat/debugger).
