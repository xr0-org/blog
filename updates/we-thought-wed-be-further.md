# #0: We thought we'd be further by now

Our update today is humbling, but hopeful. It's been 59 days since the launch,
and we are not on track with the timeline that we put forward.

_forrest gump GIF_

## What went well

In the first three-ish weeks we overhauled the underlying model that enables Xr0
to reason about memory safety. Our new model is far more faithful to the
semantics of C in the standard, and we think it capable of representing
virtually all the scenarios that arise in C programs (with respect to memory
safety). We're looking forward to documenting this model in detail (but it's
slightly undercooked).

Another thing that went well was we built a (limited!) math library that helps
us reason about bounds (in the new model) pretty cleanly.

## What went wrong

During the overhaul, as we worked towards functional parity with the previous
model, we rabbit-holed into verification of loops (something which the launched
version addressed in a very hand-wavy manner). We say "rabbit-holed" because our
"post-mortem" analysis of the way we've spent time suggests that this was the
wrong thing to focus on getting right immediately.

In particular, we've spent about five weeks on loops, and though our labour has
not been entirely fruitless, we have not (so to speak) closed the loop. There
are some deep structural issues relating to invariants that we have not been
able to adapt to our vision for Xr0.

## Looking forward

Screw loops (for now).

More seriously, we're leaving loop problem open and focusing on expanding Xr0's
representational capacities so that it can handle the Xr0 codebase.

At a high level, self-hosting Xr0 will require

1. More data types, especially structs, unions, enums and chars

2. The ability to recursively predicate about objects

3. Generation of real binaries from Xr0 `*.x` source

4. Loops (which we're ignoring for now).

For the next two weeks (until 2 Dec) we intend to focus on (1.) above, and it is
our commitment to provide an honest update on our progress by then.

Our goals for this period:

- At least 90% of the Xr0 codebase (measured in LoC) will parse

- In the presence of many syntactical exclusions, the entire parsed section of
  the code will verify, and at least one function will verify without being
  excluded.

Our stretch goals:

- The char, enum, struct and union types will be working end-to-end in a manner
  comparable to how pointers are currently working

- A sketch of how the recursion would work with structs (and other recursive
  data types) will be ready.
