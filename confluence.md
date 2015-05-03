---
title: A Proof of Church Rosser in Twelf
tags: twelf, types
---

An important property in any term rewriting system, a system of rules for saying
one term can be rewritten into another, is called confluence. In a term
rewriting system more than one rule may apply at a time, confluence states that
it doesn't matter in what order we apply these rules. In other words, there's
some sort of diamond property in our system

                    the original term
                        /      \
                       /        \
              Rule 1  /          \ Rule 2
                     /            \
                    /              \
               result 1            result 2
                    \               /
                     \             /
                      \           /
               A bunch \   of    / rules later
                        \       /
                         \     /
                     The same result

In words (and not a crappy ascii picture)

 1. Suppose we have some term `A`
 2. The system lets us rewrite `A` to `B`
 3. The system lets us rewrite `A` to `C`

Then two things hold

 1. The system lets us rewrite `B` to `D` in some number of rules
 2. The system lets us rewrite `C` to `D` with a different series of rules

In the specific case of lambda calculus, confluence is referred to as the
"Church-Rosser Theorem". This theorem has several important corollaries,
including that the normal forms of any lambda term is unique. To see this,
remember that a normal form is always "at the bottom" of diamonds like the one
we drew above. This means that if some term had multiple steps to take, they all
must converge before one of them reaches a normal form. If any of them did hit a
normal form first, they couldn't complete the diamond.

## Proving Church-Rosser

In this post I'd like to go over a proof of the Church Rosser theorem in Twelf,
everyone's favorite mechanized metalogic. To follow along if you don't know
Twelf, perhaps some shameless [self][intro] [linking][worlds] will help.

We need to start by actually defining lambda calculus. In keeping with Twelf
style, we laugh at those restricted by the bounds of inductive types and use
higher order abstract syntax to get binding for free.

``` twelf
    term : type.
    ap   : term -> term   -> term.
    lam  : (term -> term) -> term.
```

We have to constructors, `ap`, which applies one term to another. The
interesting one here is `lam` which embeds the LF function space, `term -> term`
into `term`. This actually makes sense because `term` isn't an inductive type,
just a type family with a few members. There's no underlying induction principle
with which we can derive contradictions. To be perfectly honest I'm not sure how
the proof of soundness of something like Twelf %total mechanism proceeds. If a
reader is feeling curious, I believe [this][total] is the appropriate paper to
read.

With this, something like `λx. x x` as `lam [x] ap x x`.

Now on to evaluation. We want to talk about things as a term rewriting system,
so we opt for a small step evaluation approach.

``` twelf
     step : term -> term -> type.
     step/b    : step (ap (lam F) A) (F A).
     step/ap1  : step (ap F A) (ap F' A)
                 <- step F F'.
     step/ap2  : step (ap F A) (ap F A')
                  <- step A A'.
     step/lam  : step (lam [x] M x) (lam [x] M' x)
                  <- ({x} step (M x) (M' x)).

     step* : term -> term -> type.
     step*/z : step* A A.
     step*/s : step* A C
                <- step A B
                <- step* B C.
```

We start with the 4 sorts of steps you can make in this system. 3 of them are
merely "if you can step somewhere else, you can pull the rewrite out", I've
heard these referred to as compatibility rules. This is what `ap1`, `ap2` and
`lam` do, `lam` being the most interesting since it deals with going under a
binder. Finally, the main rule is `step/b` which defines beta reduction. Note
that HOAS gives us this for free as application.

Finally, `step*` is for a series of steps. We either have no steps, or a step
followed by another series of steps. Now we want to prove a couple theorems
about our system. These are mostly the lifting of the "compatibility rules" up
to working on `step*`s. The first is the lifting of `ap1`.

``` twelf
     step*/left : step* F F' -> step* (ap F A) (ap F' A) -> type.
     %mode +{F : term} +{F' : term} +{A : term} +{In : step* F F'}
     -{Out : step* (ap F A) (ap F' A)} (step*/left In Out).

     - : step*/left step*/z step*/z.
     - : step*/left (step*/s S* S) (step*/s S'* (step/ap1 S))
          <- step*/left S* S'*.

     %worlds (lam-block) (step*/left _ _).
     %total (T) (step*/left T _).
```

The theorem says that if `F` steps to `F'` in several steps, for all `F`,
`ap F A` steps to `ap F' A` in many steps. The actual proof is quite boring,
we just recurse and apply `step/ap1` until everything type checks.

Note that the world specification for `step*/left` is a little strange. We use
the block `lam-block` because later one of our theorem needs this. The block is
just

    %block lam-block : block {x : term}.

We need to annotate this on all our theorems because Twelf's world subsumption
checker isn't convinced that `lam-block` can subsume the empty worlds we check
some of our theorems in. Ah well.

Similarly to `step*/left` there is `step*/right`. The proof is 1 character off
so I won't duplicate it.

``` twelf
    step*/right : step* A A' -> step* (ap F A) (ap F A') -> type.
````

Finally, we have `step/lam`, the lifting of the compatibility rule for
lambdas. This one is a little more fun since it actually works by pattern
matching on functions.

``` twelf
     step*/lam : ({x} step* (F x) (F' x))
                  -> step* (lam F) (lam F')
                  -> type.
     %mode step*/lam +A -B.

     - : step*/lam ([x] step*/z) step*/z.
     - : step*/lam ([x] step*/s (S* x) (S x))
          (step*/s S'* (step/lam S))
          <- step*/lam S* S'*.

     %worlds (lam-block) (step*/lam _ _).
     %total (T) (step*/lam T _).
```

What's fun here is that we're inducting on a dependent function. So the first
case matches `[x] step*/z` and the second `[x] step*/s (S* x) (S x)`. Other than
that we just use `step/lam` to lift up `S` and recurse to lift up `S*` in the
second case.

We need one final (more complicated) lemma about substitution. It states that
if `A` steps to `A'`, then `F A` steps to `F A'` in many steps for all `F`. This
proceeds by induction on the derivation that `A` steps to `A'`. First off,
here's the formal statement in Twelf

*This is the lemma that actually needs the world with `lam-block`s*

``` twelf
    subst : {F} step A A' -> step* (F A) (F A') -> type.
    %mode subst +A +B -C.
```

Now the actual proof. The first two cases are for constant functions and the
identity function

``` twelf
    - : subst ([x] A) S step*/z.
    - : subst ([x] x) S (step*/s step*/z S).
```

In the case of the constant functions the results of `F A` and `F A'` are the
same so we don't need to step at all. In the case of the identity function we
just step with the step from `A` to `A'`.

In the next case, we deal with nested lambdas.

``` twelf
     - : subst ([x] lam ([y] F y x)) S S'*
          <- ({y} subst (F y) S (S* y))
          <- step*/lam S* S'*.
```

Here we recurse, but we carefully do this under a pi type. The reason for doing
this is because we're recursing on the open body of the inner lambda. This has a
free variable and we need a pi type in order to actually apply `F` to something
to get at the body. Otherwise this just uses `step*/lam` to lift the step across
the body to the step across lambdas.

Finally, application.

``` twelf
     - : subst ([x] ap (F x) (A x)) S S*
          <- subst F S F*
          <- subst A S A*
          <- step*/left F* S1*
          <- step*/right A* S2*
          <- join S1* S2* S*.
```

This looks complicated, but isn't so bad. We first recurse, and then use various
compatibility lemmas to actually plumb the results of the recursive calls to the
right parts of the final term. Since there are two individual pieces of
stepping, one for the argument and one for the function, we use `join` to slap
them together.

With this, we've got all our lemmas

``` twelf
    %worlds (lam-block) (subst _ _ _).
    %total (T) (subst T _ _).
```

## The Main Theorem

Now that we have all the pieces in place, we're ready to state and prove
confluence. Here's our statement in Twelf

``` twelf
    confluent : step A B -> step A C -> step* B D -> step* C D -> type.
    %mode confluent +A +B -C -D.
```

Unfortunately, there's a bit of a combinatorial explosion with this. There are
approximately 3 * 3 * 3 + 1 = 10 cases for this theorem. And thanks to the
lemmas we've proven, they're all boring.

First we have the cases where `step A B` is a `step/ap1`.

``` twelf
     - : confluent (step/ap1 S1) (step/ap1 S2) S1'* S2'*
          <- confluent S1 S2 S1* S2*
          <- step*/left S1* S1'*
          <- step*/left S2* S2'*.
     - : confluent (step/ap1 S1) (step/ap2 S2)
          (step*/s step*/z (step/ap2 S2))
          (step*/s step*/z (step/ap1 S1)).
     - : confluent (step/ap1 (step/lam F) : step (ap _ A) _) step/b
          (step*/s step*/z step/b) (step*/s step*/z (F A)).
```

In the first case, we have two `ap1`s. We recurse on the smaller `S1` and `S2`
and then immediately use one of our lemmas to lift the results of the recursive
call, which step the functions in the `ap` we're looking at, to work across the
whole `ap` term.

In the second case there, we're stepping the function in one, and the argument
in the other.

## Wrap Up

[intro]: /posts/2015-02-28-twelf.html
[worlds]: /posts/2015-03-07-worlds.html
[total]: http://repository.cmu.edu/cgi/viewcontent.cgi?article=2238&context=compsci