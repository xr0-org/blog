---
published: Sep 07, 2024
author: [team]
---

# #21: Refactoring with great zeal

4D chess or obsession with details

## tl;dr

We spent the last few days refactoring. 0db (the Xr0 debugger) now has a test
harness. The Makefile is substantially simplified. The modules that previously
controlled progress through verification (path, state, stack, program) have been
refactored into a slightly different set (verifier, path, segment, state, stack,
program) that better reflects what Xr0 does and reduces the coupling of
different layers of abstraction.

## Some details

### 0db’s test harness

Although we are uncertain about the long-term usefulness of 0db, except as a
tool for understanding how Xr0 works, it is definitely of great help to us in
debugging Xr0. Frequently we find that we break 0db in subtle ways, so that
programs which pass verification may no longer work in the debugger.

We’ve put together a debugger-testing script which takes directories, each
containing a list of debugger instructions and expected outputs, and uses them
to raise the alarm whenever 0db behaves differently than we’ve previously
expected. This will give us more confidence in making changes as we go forward.

### Make

Our [previous
Makefile](https://github.com/xr0-org/xr0/blob/4b1bec8aee7118fc3aa0e9b0afafd1963ac88e03/Makefile)
was a naive list of the source files in Xr0 with their several
interdependencies. The difficulty is this list was getting distractingly (and
embarrassingly) long, and given C’s limited features for modularisation it was
becoming a real inconvenience. Adding a source file meant adding a new target to
the long line of objects to be built, and adding a module meant doing [even
uglier things than
that](https://github.com/xr0-org/xr0/blob/4b1bec8aee7118fc3aa0e9b0afafd1963ac88e03/src/ast/ast.c#L11).

Inspired by the approach taken in
[lacc](https://github.com/larmel/lacc/blob/master/configure#L133), we’ve written
a `./configure`
[script](https://github.com/xr0-org/xr0/blob/3f2abe49f9ea57e0ec81ce26c3fdd249480e7ea2/configure)
that generates the (new) Makefile from a [much-simpler
template](https://github.com/xr0-org/xr0/blob/3f2abe49f9ea57e0ec81ce26c3fdd249480e7ea2/scripts/tmpl.mk).

The script walks through the `/src` of the repo, treating each directory
(together with its descendant directories) as a module that can access all the
headers in any folder named `include` within the same directory. Thus files in
`/src` and `/src/ast` and `/src/ast/expr` can all see the headers in
`/src/include`, files in `/src/ast` and `/src/ast/expr` can use the headers in
`/src/ast/include`, and files in `/src/ast/expr` can also use headers in
`/src/ast/expr/include`. Apart from this each file is treated as its own
translation unit. (You can read the all the rules the script follows
[here](https://github.com/xr0-org/xr0/blob/3f2abe49f9ea57e0ec81ce26c3fdd249480e7ea2/configure#L9).)

The best side-effect of introducing this script is we no longer have to hesitate
when modularising or splitting functionality into separate files. We could have
done this a lot earlier but we were probably too insecure to write something
this naive, thinking that “real C programmers” have better ways for doing these
things. But we’re learning that real C programmers (evidenced e.g. from lacc’s
build process) are happy to be naive — they have outgrown the “maturity” of
sophistication, apart from the sophistication of simplicity and directness.

### Refactoring path and state

These are perhaps the most important yet the most difficult-to-explain changes
we’ve made during this iteration. In essence what we’ve done is concentrated
upon some of the compromises of [Dijkstra’s rules of
abstraction](https://www.cs.utexas.edu/~EWD/transcriptions/EWD02xx/EWD273.html)
that existed between the path and state modules and through a series of
thoughtful modifications reduced the coupling and clarified the code. Time will
prove how much insight we have derived in doing this, but we have a very good
feeling about things so far.

## Next up

We enter the next iteration with a simpler codebase but focused on the same
goals: [setup verification of bounds](https://xr0.dev/learn#avoiding-double-frees-with-setups)
and wrapping up [the local-constant case](https://github.com/xr0-org/xr0/issues/58) of buffer
out-of-bounds prevention.

See you on 21st.
