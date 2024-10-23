---
published: Jan 27, 2024
author: [team]
---

# #11: One Last thing?

We achieved our goal ([#24](https://github.com/xr0-org/xr0/issues/24)) from last
week of verifying the case where the value of one of the returned fields of a
returned structure determines whether or not an allocation will take place. 

We now have one function left to verify in
[parse.x](https://github.com/xr0-org/xr0/blob/9a65d5ecd5873dcd8e2fce5d1169630bdf5c25d1/tests/3-program/100-lex/parse.x),
`main`. The challenge we’ve run into with it is evaluating all the scenarios
that can occur due to the complexity of [parse’s
abstract](https://github.com/xr0-org/xr0/blob/feat/advanced-branch/tests/3-program/100-lex/parse.x#L40)[^abstract].
Xr0 handles selection statements by generating all the possible execution paths
given a function’s abstract and body. For example, from

  [^abstract]: An *abstract* is the semantic summary of a function’s behaviour.
  See here for a fuller definition.

```
void *
foo(int x)
{
    if (x) {
        return malloc(1);
    }
    return NULL;
} 
```

it identifies two paths corresponding to whether the body of the if-statement is
entered. The problem is that until now this “splitting” process has been
entirely syntactical, because this allowed us to generate all the possible paths
through a function at the beginning of our verification procedure.

With [the way that main calls
parse](https://github.com/xr0-org/xr0/blob/9a65d5ecd5873dcd8e2fce5d1169630bdf5c25d1/tests/3-program/100-lex/parse.x#L94),
however, we have to account for the possibilities that come from the various
selection statements in parse’s abstract. This can plainly no longer be
syntactical only, because at the very least we have to move from the call

```
l = parse(file);
```

to a consideration of parse’s abstract. We have therefore been reworking the
“splitting” process, integrating it into the verification procedure. In the new
approach we’ve taken the various paths are generated whenever we encounter a
line of code that is splitting, such as a selection statement but also any calls
of functions with selection statements in their abstracts.

We will explain this in more detail soon, but for now it’s back to building.

## Next up

Our goal for this week (to 3rd) is getting the new splitting procedure to work
and establishing whether or not it will help us verify main
[[#25](https://github.com/xr0-org/xr0/issues/25)].

Stretch goal:
    - Simplify map and array data structures [[#13](https://todo.sr.ht/~lbnz/xr0/13)].
