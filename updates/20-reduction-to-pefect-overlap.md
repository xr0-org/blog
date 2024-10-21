---
published: Aug 28, 2024
author: [team]
---

# #20: Reduction to perfect overlap

## tl;dr

We implemented all the relational (`<`, `<=`, `>=`, `>`) operators for the
special case that the two rconsts[^rconst] have ranges overlapping in at most
discontiguous points. This is done using an algorithm that reduces comparisons
between such rconsts to trivial cases. We also converged our equality operators
(`==`, `!=`) to rely on this same algorithm. We will spend the next week or so
doing some refactoring, as well as applying these operators to concluding on the
[local-constant size][lcs] case of buffer out-of-bounds prevention.

  [^rconst]: An _rconst_ (runtime-constant) is our representation of a value
  that is fixed at runtime. It allows us to reason strictly about what code will
  do in the presence of uncertainty about the runtime value. It is explained
  more fully in the [Rconsts](#rconsts) section. 

  [lcs]: https://github.com/xr0-org/xr0/issues/58

## Rconsts

As we explained previously, protection from out-of-bounds errors comes down to
tracking what is known about the possible values that the size of the buffer and
the accessed index (or dereferenced pointer) can take on at runtime. We handle
this with what we call _rconsts_. An _rconst_ (runtime-constant) is our
representation of a value that is fixed at runtime. For example, in the code
below,

```C
f(int x) { /* do stuff */ }
```

the value of `x` could be any integer, so we regard it as an rconst taking on
values in the interval from `INT_MIN` to `INT_MAX`. We use the notation `[a?b]`
to express the half-open range of values `n` such that `a ≤ n < b`, but when `a`
is `INT_MIN` or `b` is `INT_MAX` we leave it out, so that `[?]` is the range of
values that C integers can take on.
(We only regard `[a?b]` as a range if `a < b`.)

In the abstract the problem of comparing rconsts is completely intractable.
Consider what happens if two integers `x` and `y` take on rconst values in
`[?]`. The value of any given comparison between them, say `x < y`, depends upon

$$
(\mathrm{INT\_MAX}-\mathrm{INT\_MIN})^2 = 2^{32} \approx 4
$$

billion different pairings. An intuitive sense for the hopelessness of progress
in such a situation is probably why most programmers doubt the feasibility of
formal verification.

Because we are not wizards, our analysis in Xr0 always and always depends upon
the assumption that the programmer can reduce this impossible calculation to
manageable special cases. As this relates to out-of-bounds protection, this
means when we compare two rconsts, our assumption is that programmer-imposed
constraints on the state make it such that the set of possibilities to consider
is comparably small. Last week we showed an example of this:

```C
bar(int i) ~ [ setup: i = [0?5]; ]                                               
{                                                                                
        int arr[2];                                                              
        if (i < 2) {                                                             
                arr[i] = 0;                                                      
        }                                                                        
}
```

Because the buffer size [fixed] as `2`, the question of whether `arr[i]` is an
out-of-bounds error is in effect two comparisons, `0 ≤ i` and `i < 2`, both of
which involve only one non-constant term. Even in [more general cases][general],
the underlying idea remains that the programmer reduces the number of cases by
giving us some sort of relation between the rconsts under comparison. So our
task in Xr0 is not to solve the impossible abstract comparison problem, but
rather, to offer a set of tools that facilitate verification about the concrete,
manageable scenarios that actually arise in programs.
Thus by "[the general case][general]" we mean "the general _manageable_ case".

  [fixed]: https://github.com/xr0-org/xr0/issues/58
  [general]: https://github.com/xr0-org/xr0/issues/60

What we did this week is identify a cluster of special cases (for such
comparisons) that function as an ideal stepping stone towards solving this
general manageable case. This is when we have rconsts whose underlying ranges
overlap in discontiguous points only. For example, in both `0 ≤ i` and `i < 2`,
because `0` and `2` are single points, they overlap with `i` in at most a single
point. If we were to compare two rconsts in `[0?2]`, however, we would have two
contiguous points, `0` and `1`, of overlap.

## Reduction to perfect overlap

To explain why this case, which we shall call the _separable rconsts_ case, is
easy to deal with, we have to describe the envisioned solution to the general
manageable case.

**Definition.**
A _range_ of values `[a?b]` is the set of integers `x` satisfying `a ≤ x < b`.
We include in this definition the requirement that `a < b`.

**Definition.**
The set `X` of values that an integer `x` can take at any sequence point[^sp]
in a program is called its _possibility set_. The tuple `(x, X)` is called an
_rconst_.

  [^sp]: This is a Standard-defined concept:
    > At certain specified points in the execution sequence called sequence
    > points, all side effects of previous evaluations shall be complete and no
    > side effects of subsequent evaluations shall have taken place. 

We will generally think of the possibility set as the union of disjoint ranges
since we can always reduce a subset of the integers in `[?]` to such a form (in
the extreme using each included integer `m` as one range `[m?m+1]` in the
union).

We can answer questions involving comparison[^comp] operators if we can put the
rconsts into _[partial order]_, so that one precedes the other. The question of
whether or not this order will be strict (resembling `<`) or not (`≤`) is not
important, but because we chose half-open ranges, a strict order is preferable.
Now the first important observation is that we can define a strict partial order
on any two ranges whenever there is no overlap between them:

  [^comp]: The Standard distinguishes between _[relational]_ and _[equality]_
  operators to secure the weaker binding strength of the latter; here we use the
  term _comparison_ to include both of them.

  [relational]: https://port70.net/~nsz/c/c89/c89-draft.html#3.3.8
  [equality]:   https://port70.net/~nsz/c/c89/c89-draft.html#3.3.9

  [partial order]: https://en.wikipedia.org/wiki/Partially_ordered_set


**Definition.**
Let `[a?b]` and `[c?d]` be two ranges. We shall say `[a?b]` is _less than_
`[c?d]` and write `[a?b] < [c?d]` if `b ≤ c`.

Less-than is a _[strict partial order][spo]_ on the set of mutually-disjoint
ranges:

1. _Irreflexive_: We always have `b = b`, so `[a?b] < [a?b]` is false.

2. _Asymmetric_: Suppose `[a?b] < [c?d]` is false. Then `b > c`. If `d > a` we
   must have `a ≤ c`, since our definition or ranges requires `c < d`. But then
   `a ≤ c < b`, because `c < b` or `b > c`. This contradicts means `c` is in
   `[a?b]`, which contradicts the disjointedness of `[a?b]` and `[c?d]`. 
   Thus `d > a` cannot be, so we have `a ≤ d`: `[c?d] < [a?b]`.

3. _Transitive_: This one is trivial and follows from the transitivity of `≤` on
   integers.

  [spo]: https://en.wikipedia.org/wiki/Partially_ordered_set#Strict_partial_orders

The second observation is that we can reduce any two rconsts, by means of
splitting[^splitting] the state, to scenarios that involve either _equality_ or
_mutual disjointedness_ of their possibility sets. Let us see why this is the
case. We begin with two arbitrary rconsts `(x, X)` and `(y, Y)`.

  [^splitting]: _Splitting_ refers to generating from one state a series of
  states that cover all the possible scenarios in the original. This helps us
  make certain statements decidable, like in `bar` above creating a scenario in
  which `i < 2` and one in which it isn't.

We may assume, without any loss of generality, that `X` and `Y` are individual
ranges `[a?b]` and `[c?d]`, because if this isn't the case we can split the
state into every pairing of the ranges that constitute them. I.e. we will have

$$
X = \cup\{[a_0?b_0], \ldots, [a_{m-1}, b_{m-1}]\}
$$

and

$$
Y = \cup\{[c_0?c_0], \ldots, [c_{n-1}, c_{n-1}]\}
$$

so we can separately consider the `ij` cases where we have

$$
X = [a_i?b_i], Y = [c_j?d_j]
$$

for $0\le i \le m$, $0 \le j \le n$.
That $ij$ is a small number follows from the fact that no restrictions the
programmer places on the state will produce a large enough `m` or `n`, at least
where our concern is bounds.[^bounds]

  [^bounds]: When we come to consider multiplication, the effect of multiplying
  an rconst in `[0?5]` by `5` would produce a "periodic" set involving `0`, `5`,
  `10` and `20`. This is an important scenario that will require some analysis
  of its own, but if our concern is to analyse whether such an interval will
  overshoot some bound it is safe to think of it as `[0?25]`.

The second assumption we can make is that `a ≤ c`. If the case is otherwise we
can simply re-name and call `X` `[c?d]` and `Y` `[a?b]`.

The third assumption we make is more surprising: that `a = c`. This is possible
because if `a < c`, we can "cut off" the case where `X = [a?c], Y = [c?d]`,
which is a case of mutual disjointedness. With this case analysed, the only
possibilities that remain are where `X = [c?b], Y = [c?d]`, which share the same
lower bound (so that we can rename again and get `a = c`).

In like fashion, we can cut off the longer tail and get the assumption that
`b == d`.
This kind of cutting thus reduces an arbitrary pair of rconsts to a situation
where the two possibility sets are either disjoint or overlap exactly.

As we showed above, the case of disjointedness allows us to order these ranges
easily, so that we can answer any comparison such as `x == y` or `x < y`. In
addition, if there is exact overlap in a single point, we can answer that
`x == y` is true (and every comparison that includes/excludes equality is
true/false accordingly).

What we did this week is implement the _reduction to perfect overlap_ algorithm,
so that now the only situation that we cannot contend with is that where the
overlap involves more than one contiguous point, which is the next (and
hopefully final) hurdle for our solving of the buffer out-of-bounds problem.

## Next up

Our plan is to spend the next iteration finishing up the 
[local-constant size][lcs] case of buffer out-of-bounds verification. At present
the internal verification of functions is working, but there is no
[setup verification][setupv] of bounds at the point of callers. We'll also be
doing a bit of refactoring.

  [setupv]: https://xr0.dev/learn#avoiding-double-frees-with-setups

We expect to update by the 7th (of next month).
