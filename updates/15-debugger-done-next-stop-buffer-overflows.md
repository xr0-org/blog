---
published: Jun 01, 2024
author: [team]
---

# #15: Debugger done. Next stop: buffer overflows

## tl;dr

We completed the first version of our static debugger. We are working on solving
buffer overflows.

## 0db: A static debugger for C

It has taken us the last two months or so to implement 0db. 0db is the
compile-time analogue of runtime debuggers like
[GDB](https://www.sourceware.org/gdb/). While runtime debuggers require you to
execute a program in order to analyse it, 0db shows line-by-line the state of a
C program on the basis of the semantics of the language alone.

This is significant because at runtime we interact with a very tiny subset of a
program’s possible behaviours. In the state supplied by 0db you see at once what
applies to all possible executions of a program.

Learn more about how 0db works on our
[website](https://xr0blog.substack.com/p/13-were-making-xr0-a-static-debugger)
and [try it out](https://xr0.dev/try) for yourself.

## Next up

The next thing on the [roadmap](https://xr0.dev/vision-roadmap) for Xr0 is
solving buffer overflows, which we’re tracking under
[[#55](https://github.com/xr0-org/xr0/issues/55)]. Currently we do not track the
sizes of objects within our model of C’s semantics. We don’t yet have enough
clarity to propose dates for this, but will post an update as soon as we do.
