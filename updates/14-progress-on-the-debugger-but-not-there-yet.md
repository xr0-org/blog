---
published: May 06, 2024
---

# #14: Progress on the debugger but not there yet

## tl;dr

We’ve made steady progress on the debugger we described in our
[last update](/updates/13-were-making-xr0-into-a-static-debugger),
but are still not yet ready to launch it. So we’re pushing the deadline for the
launch ([#45](https://github.com/xr0-org/xr0/issues/45)) up from 4th to the
18th.

## Copernican revolution in Xr0

In terms of the progress we can say a few more words. Adapting Xr0 to support
the debugger involved a Copernican revolution of sorts in the way our
verification routine moves through a program. Previously it was AST-based,
taking a function as a series of (AST) statements and working through these
one-by-one. This seems amenable to user control by adding stepping control to
the loop that works through these statements.

The challenge comes in when handling statements involving function calls,
because these require a call stack for the called function and the ability to
step through its statements. This is especially the case when there is a
selection statement in the called function that requires a “state split” — a
breaking of the state into different paths based on the possible values of the
controlling expression of the selection statement. If we stick to an AST-based
model, it’s impossible to do the split without “rewinding” to the beginning of
the call in question, because the value of the controlling expression will
depend on the arguments supplied to the function, which belong to the state of
the function being verified.

So what we’ve done is rework the way Xr0 verifies functions entirely. Instead of
anchoring analysis on a direct traversal of the AST, Xr0 now simulates progress
through a function as progress through a stack machine of sorts. This machine
has a call stack that has a program counter and block of code for each stack
frame. Motion through a function is then handled by loading the function’s body
as the block for the lowest stack frame, and the abstract of each called
function comes as a higher frame. This model, though harder to grasp at first,
localises in one place (the stack machine) the entire stack trace with
corresponding program counters, so we have coordinates for where we are no
matter how far up a call stack we are. This makes it awfully convenient for
doing state splits and stack traces, and also allows us to implement the
debugger through adding user controls for the stack machine.

## Next up

That’s all we’ll say for now. We realise how high level and dense the paragraphs
above must seem, and will be documenting these things in much greater detail
once the debugger is working beautifully. See you on May 18th — or earlier
