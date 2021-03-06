---
title: A Basic Tutorial on JonPRL
tags: jonprl, types
---

I was just over at OPLSS for the last two weeks. While there I finally
met [Jon Stering][jon] in person. What was particularly fun is that
for that last few months he's been creating a theorem prover called
[JonPRL][jonprl] in the spirit of NuPRL.

As it turns out, it's quite a fun project to work on so I've
implemented a few features in it over the last couple of days and
learned more or less how it works.

Since there's basically no documentation on it besides the readme and
of course the compiler I thought I'd write down some of the stuff I've
learned. There's also a completely separate post on the underlying
type theory for NuPRL and JonPRL that's very interesting in its own
right but this isn't it. Hopefully I'll get around to scribbling
something about that because it's really quite clever.

## Getting JonPRL

JonPRL is pretty easy to build and install. You'll need `smlnj` since
JonPRL is currently written in SML. This is available in most package
managers (including homebrew I've heard?) otherwise just grab the
binary from the [website][nj]. After this the following commands
should get you a working executable

 - `git clone ssh://git@github.com/jonsterling/jonprl`
 - `cd jonprl`
 - `git submodule init`
 - `git submodule update`
 - `make`
 - `make test` (If you're doubtful)

You should now have an executable called `jonprl` in the `bin`
folder. There's no prelude for jonprl so that's it. You can now just
feed it files like any reasonable compiler and watch it spew
(currently meaningless) output at you.

If you're interested in actually writing JonPRL code, you should
probably install David Christiansen's [mode][mode]. Now that we're up
and running, let's actually figure out how the language works

## The Different Languages in JonPRL

JonPRL is composed of really 3 different sorts of constructs

 - The term language
 - The tactic language
 - The language of commands to the theorem prover

In Coq, these roughly correspond to Gallina, Ltac, and Vernacular
respectively.

### The Term Language

The term language is an untyped language that contains a number of
constructs that should be familiar to people who have been exposed to
dependent types before. The actual concrete syntax is composed of 3
basic constructs. We can apply an "operator" (I'll clarify this in a
moment) with `op(arg1; arg2; arg3)`, we have variables with `x`, and
we have abstraction with `x.e`.

An operator in this context is really anything you can imagine having
a node in an AST for a language. So something like λ is an operator,
as is `if` or `pair` (corresponding to `(,)` in Haskell). Each
operator has a piece of information associated with it, called its
arity. This arity tells you how many arguments an operator takes and
how many variables `x.y.z. ...` each is allowed to bind. For example,
with λ has an arity is written `(1)` since it takes 1 argument which
binds 1 variable. Application (`ap`) has the arity `(0; 0)`. It takes
2 arguments neither of which bind a variable.

So as mentioned we have functions and application. This means we could
write `(λ x → x) y` in JonPRL as `ap(λ(x.x); y)`. The type of
functions is written with `Π`. Remember that JonPRL's language has a
notion of dependence so the arity is `(0; 1)`. The construct `Π(A;
x.B)` corresponds to `(x : A) → B` in Agda or `forall (x : A), B` in
Coq.

We also have dependent sums as well (`Σ`s). In Agda you would write
`(M , N)` to introduce a pair and `Σ A λ x → B` to type it. In JonPRL
you have `pair(M; N)` and `Σ(A; x.B)`. To use a `Σ` we have spread
which let's us provide a term which depends on the two components of a
pair. Eg `spread(0; 2)`, you give it a `Σ` in the first spot and
`x.y.e` in the second and it'll replace `x` with the first component
and `y` with the second. Can you think of how to write `fst` and `snd`
with this?

There's sums, so `inl(M)`, `inr(N)` and `sum+(A; B)` corresponds to
`Left`, `Right`, and `Either` in Haskell. For case analysis there's
`decide` which has the arity `(0; 1; 1)`. You should read `decide(M;
x.N; y.P)` as something like

``` haskell
    case E of
      Left x -> L
      Right y -> R
```

In addition we have `unit` and `<>` (pronounced axe for axiom
usually). Neither of these takes any arguments so we write them just
as I have above. They correspond to `()` and `()` in Haskell. Finally
there's `void` which is sometimes called `false` or `empty` in theorem
prover land.

You'll notice that I presented a bunch of types as if they were normal
terms in this section. That's because in this untyped computation
system, types *are literally* just terms. There's no typing relation
to distinguish them yet so they just float around exactly as if they
were λ or something! I call them types because I'm thinking of later
when we have a typing relation built on top of this system but for now
there are really just terms. Nothing more, nothing less. This
justifies introducing several more exotic terms into our language

 - `U{i}`, the ith level universe. This should be thought of as the
   type of types but we don't have types yet and it's not really the
   type of all types. All metaphors suck eventually *shrug*.
 - `=(0; 0; 0)` this is equality between two terms at a type. It's a
   proposition that's going to precisely mirror what's going on later
   in the type theory with the equality judgment
 - `∈(0; 0)` this is just like `=` but internalizes membership in a
   type into the system. Remember that normally "This has that type"
   is a judgment but with this term we're going to have a
   propositional counterpart to use in theorems.

In particular it's important to distinguish the difference between `∈`
the judgment and ∈ the term. There's nothing inherent in `∈` above
that makes it behave like a typing relation as you might
expect. Later, we're going to construct some rules around `∈` that are
going to make it behave that way but for now, it and `=` are just
suggestively named constants.

This term language contains the full untyped lambda calculus so we can
write all sorts of fun programs like

``` jonprl
    λ(f.ap(λ(x.ap(f;(ap(x;x)))); λ(x.ap(f;(ap(x;x)))))
```

which is just the Y combinator. In particular this means that there's
no reason that every term in this language should normalize to a
value. There are plenty of terms in here that diverge and in
principle, there's nothing that rules out them doing even stranger
things than that. We really only depend on them being deterministic,
that `e ⇒ v` and `e ⇒ v'` implies that `v = v'`.

### Tactics

The other big language in JonPRL is the language of tactics. Luckily,
this is very familiarly territory if you're a Coq user. Unluckily, if
you've never heard of Coq's tactic mechanism this will seem completely
alien. As a quick high level idea for what tactics are:

When we're proving something in a theorem prover we have to deal with
a lot of boring mechanical details. For example, when proving
`A → B → A` I have to describe that I want to introduce the `A` and
the `B` into my context, then I have to suggest using that `A` the
context as a solution to the goal. Bleh. All of that is pretty obvious
so let's just get the computer to do it! In fact, we can build up a
DSL of composable "proof procedures" or tactics to modify a particular
goal we're trying to prove so that we don't have to think so much
about the low level details of the proof being generated. In the end
this DSL will generate a proof term (or derivation in JonPRL) and
we'll check that so we never have to trust the actual tactics to be
sound.

In Coq this is used to great effect. In particular see Adam Chlipala's
[book][cpdt] to see incredibly complex theorems with one line proofs
thanks to tactics.

In JonPRL the tactic system works by modifying a sequent of the form
`H ⊢ A` (a goal). Each time we run a tactic we get back a list of new
goals to prove until eventually we get to trivial goals which produce
no new subgoals. This means that when trying to prove a theorem in the
tactic language we never actually see the resulting extract, the
program to be extracted from our proof. We just see this list of
`H ⊢ A`s to prove and we do so with tactics.

The tactic system is quite simple, to start we have a number of basic
tactics which are useful no matter what goal you're attempting to
prove

 - `id` a tactic which does nothing
 - `t1; t2` this runs the `t1` tactic and runs `t2` on any resulting
    subgoals
 - `*{t}` this runs `t` as long as `t` does *something* to the
   goal. If `t` ever fails for whatever reason it merely stops
   running, it doesn't fail itself
 - `?{t}` tries to run `t` once. If `t` fails nothing happens
 - `!{t}` runs `t` and if `t` does anything besides complete the proof
   it fails. This means that `!{id}` for example will always fail.
 - `t1 | t2` runs `t1` and if it fails it runs `t2`. Only one of the
    effects for `t1` and `t2` will be shown.
 - `[t1, ..., tn]` runs tactic `ti` on subgoal `i`
 - `trace "some words"` will print `some words` to standard out. This
   is useful when trying to figure out why things haven't gone your
   way.
 - `fail` is the opposite of `id`, it just fails. This is actually
   quite useful for forcing backtracking and one could probably
   implement a makeshift `!{}` as `t; fail`.

Now those give us a sort of bedrock for building up scripts of
tactics. We also have a bunch of tactics that actually let us
manipulate things we're trying to prove. The 4 big ones to be aware of
are

 - `intro`
 - `elim #NUM`
 - `eq-cd`
 - `mem-cd`

The basic idea is that `intro` modifies the `A` part of the goal. If
we're looking at a function, so something like  `H ⊢ Π(A; x.B)`, this
will move that `A` into the context, leaving us with
`H, x : A ⊢ B`.

If you're familiar with sequent calculus at all `intro` runs the
appropriate right rule for the goal. If you're not familiar with
sequent calculus `intro` looks at the outermost operator of the `A`
and runs a rule that applies when that operator is to the right of a
the `⊢`.

Now one tricky case is what should `intro` do if you're looking at a
Σ? Well now things get a bit tricky. Intuitively we'd expect to get
two subgoals if we run `intro` on `H ⊢ Σ(A; x.B)`, one which proves `H
⊢ A` and one which proves `H ⊢ B` or something, but what about the
fact that `x.B` *depends* on whatever the underlying realizer
(remember that's the program extracted from) the proof of `H ⊢ A`!
Even more troubling, NuPRL and JonPRL are based around extract-style
proof systems which shouldn't allow a goal to depend on the particular
content of a realizer of another goal. So instead we have to tell
`intro` up front what we want the realizer for `H ⊢ A` to be is.

To do this we just give intro an argument. For example say we're
proving that `· ⊢ Σ(unit; x.unit)`, we run `intro [<>]` which gives us
two subgoals `· ⊢ ∈(<>; unit)` and `· ⊢ unit`. Here the `[]` let us
denote the realizer we're passing to `intro`, in general any term
arguments to a tactic will be wrapped in `[]`s. So the first goals
says "OK, you said that this was your realizer for `unit`, but is it
actually a realizer for `unit`?" and the second one substitutes the
given realizer into the second argument of `Σ`, `x.unit` and asks us
to prove that. Notice how here we have to prove `∈(<>; unit)`? This is
where that weird `∈` type comes in handy. It let's us sort of play
type checker and guide JonPRL through the process of type
checking. This is actually very crucial since type checking in NuPRL
and JonPRL is undecidable.

Now how do we actually go about proving `∈(<>; unit)`? Well here
`mem-cd` has got our back. This tactic transforms `∈(A; B)` into the
equivalent form `=(A; A; B)`. In JonPRL and NuPRL, types are given
meaning by how we interpret the equality of their members. In other
words, if you give me a type you have to say

 1. What it means to be in that type
 2. What it means for two members to be equal

Long ago, Stuart Allen realized we could combine the two by specifying
a *partial* equivalence relation for a type. In this case rather than
having a separate notion of membership we just check to see if
something is equal to itself under the PER because when it is that PER
behaves like a normal equivalence relation! So in JonPRL ∈ is actually
just a very thin layer of sugar around `=` which is really the core
defining notion of typehood. To handle `=` we have `eq-cd` which does
clever things to handle most of the obvious cases of equality.

Finally, we have `elim`. Just like `intro` let us simplify things on
the right of the ⊢, `elim` let's us eliminate something on the
left. So we tell `elim` to "eliminate" the nth item in the context
(they're numbered when JonPRL prints them) with `elim #n`.

Just like with anything, it's hard to learn all the tactics without
experimenting (though a complete list can be found with
`jonprl --list-tactics`). Let's go look at the command language so we
can actually prove some theorems.

### Commands

So in JonPRL there are only 4 commands you can write at the top level

 - `Operator`
 - `[oper] =def= [term]` (A definition)
 - `Tactic`
 - `Theorem`

The first three of these let us customize and extend the basic suite
of operators and tactics JonPRL comes with. The last actually lets us
state and prove theorems.

The best way to see these things is by example so we're going to build
up a small development in JonPRL. We're going to show that products
are monoid with `unit` as the identity.

First we want a new snazzy operator to signify nondependent products
since writing `Σ(A; x.B)` is kind of annoying. We do this using
operator

``` jonprl
    Operator prod : (0; 0).
```

This line declares `x` as a new operator which takes two arguments
binding zero variables each. Now we really want JonPRL to know that
`prod` is sugar for `Σ`. To do this we use `=def=` which gives us a way
to desugar a new operator into a mess of existing ones.

``` jonprl
    [prod(A; B)] =def= [Σ(A; _.B)].
```

Now we can change any occurrence of `prod(A; B)` for `Σ(A; _.B)` as we'd
like. Okay, so we want to prove that we have a monoid here. What's the
first step? Let's verify that `unit` is a left identity for `x`. This
entails proving that for all types `A`, `prod(unit; A) ⊃ A` and
`A ⊃ prod(unit; A)`. Let's prove these as separate theorems. Translating
our first thing into JonPRL we want to prove

``` jonprl
    Π(U{i}; A.
    Π(prod(unit; A); _.
    A))
```

In Agda notation this would be written

``` agda
    (A : Set) → (_ : prod(unit; A)) → A
```

Let's prove our first theorem, we start by writing

``` jonprl
    Theorem left-id1 :
      [Π(U{i}; A.
       Π(prod(unit; A); _.
       A))] {
      id
    }.
```

This is the basic form of a theorem in JonPRL, a name, a term to
prove, and a tactic script. Here we have `id` as a tactic script,
which clearly doesn't prove our goal. When we run JonPRL on this file
(C-c C-l if you're in Emacs) you get back

    [XXX.jonprl:8.3-9.1]: tactic 'COMPLETE' failed with goal:
    ⊢ ΠA ∈ U{i}. (prod(unit; A)) => A

    Remaining subgoals:
    ⊢ ΠA ∈ U{i}. (prod(unit; A)) => A

So focus on that `Remaining subgoals` bit, that's what we have left to
prove, it's our current goal. There's nothing to the left of that ⊢
yet so let's run the only applicable tactic we know. Delete that `id`
and replace it with

``` jonprl
    {
      intro
    }.
```

The goal now becomes

Remaining subgoals:

    1. A : U{i}
    ⊢ (prod(unit; A)) => A

    ⊢ U{i} ∈ U{i'}


Two ⊢s means two subgoals now. One looks pretty obvious, `U{i'}` is
just the universe above `U{i}` (so that's like Set₁ in Agda) so it
should be the case that `U{i} ∈ U{i'}` by definition! So the next
tactic should be something like `[???, mem-cd; eq-cd]`. Now what
should that ??? be? Well we can't use `elim` because there's one thing
in the context now (`A : U{i}`), but it doesn't help us
really. Instead let's run `unfold <prod>`. This is a new tactic that's
going to replace that `x` with the definition that we wrote earlier.

``` jonprl
    {
      intro; [unfold <prod>, mem-cd; eq-cd]
    }
```

This gives us

    Remaining subgoals:

    1. A : U{i}
    ⊢ (unit × A) => A


We run intro again

``` jonprl
    {
      intro; [unfold <prod>, mem-cd; eq-cd]; intro
    }
```

Now we are in a similar position to before with two subgoals.

``` jonprl
    Remaining subgoals:

    1. A : U{i}
    2. _ : unit × A
    ⊢ A


    1. A : U{i}
    ⊢ unit × A ∈ U{i}
```

The first subgoal is really what we want to be proving so let's put a
pin in that momentarily. Let's get rid of that second subgoal with a
new helpful tactic called `auto`. It runs `eq-cd`, `mem-cd` and
`intro` repeatedly and is built to take care of boring goals just like
this!

``` jonprl
    {
      intro; [unfold <prod>, mem-cd; eq-cd]; intro; [id, auto]
    }
```

Notice that we used what is a pretty common pattern in JonPRL, to work
on one subgoal at a time we use `[]`'s and `id`s everywhere except
where we want to do work, in this case the second subgoal.

Now we have

    Remaining subgoals:

    1. A : U{i}
    2. _ : unit × A
    ⊢ A


Cool! Having a pair of `unit × A` really ought to mean that we have an
`A` so we can use `elim` to get access to it.


``` jonprl
    {
      intro; [unfold <prod>, mem-cd; eq-cd]; intro; [id, auto];
      elim #2
    }
```

This gives us

    Remaining subgoals:

    1. A : U{i}
    2. _ : unit × A
    3. s : unit
    4. t : A
    ⊢ A

We've really got the answer now, #4 is precisely our goal. For this
situations there's `assumption` which is just a tactic which succeeds
if what we're trying to prove is in our context already. This will
complete our proof

``` jonprl
    Theorem left-id1 :
      [Π(U{i}; A.
       Π(prod(unit; A); _.
       A))] {
      intro; [unfold <prod>, mem-cd; eq-cd]; intro; [id, auto];
      elim #2; assumption
    }.
```

Now we know that `auto` will run all of the tactics on the first line
except `unfold <prod>`, so what we just `unfold <prod>` first and run
`auto`? It ought to do all the same stuff.. Indeed we can shorten our
whole proof to `unfold <prod>; auto; elim #2; assumption`. With this more
heavily automated proof, proving our next theorem follows easily.

``` jonprl
    Theorem left-id2 :
      [Π(U{i}; A.
       Π(prod(A; unit); _.
       A))] {
      unfold <prod>; auto; elim #2; assumption
    }.
```

Finally we have to prove associativity to complete the development
that `x` is a monoid. The statement here is a bit more complex.

``` jonprl
    Theorem assoc :
      [Π(U{i}; A.
       Π(U{i}; B.
       Π(U{i}; C.
       Π(prod(A; prod(B;C)); _.
       prod(prod(A;B); C)))))] {
      id
    }.
```

In Agda notation what I've written above is

``` agda
    assoc : (A B C : Set) → A × (B × C) → (A × B) × C
    assoc = ?
```

Let's kick things off with `unfold <prod>; auto` to deal with all the
boring stuff we had last time. In fact, since `x` appears in several
nested places we'd have to run `unfold` quite a few times. Let's just
shorthand all of those invocations into `*{unfold <prod>}`

``` jonprl
    {
      *{unfold <prod>}; auto
    }
```

This leaves us with the state

Remaining subgoals:

    1. A : U{i}
    2. B : U{i}
    3. C : U{i}
    4. _ : A × B × C
    ⊢ A


    1. A : U{i}
    2. B : U{i}
    3. C : U{i}
    4. _ : A × B × C
    ⊢ B


    1. A : U{i}
    2. B : U{i}
    3. C : U{i}
    4. _ : A × B × C
    ⊢ C

In each of those goals we need to take apart the 4th hypothesis so
let's do that

``` jonprl
    {
      *{unfold <prod>}; auto; elim #4
    }
```

This leaves us with 3 subgoals still

    1. A : U{i}
    2. B : U{i}
    3. C : U{i}
    4. _ : A × B × C
    5. s : A
    6. t : B × C
    ⊢ A


    1. A : U{i}
    2. B : U{i}
    3. C : U{i}
    4. _ : A × B × C
    5. s : A
    6. t : B × C
    ⊢ B


    1. A : U{i}
    2. B : U{i}
    3. C : U{i}
    4. _ : A × B × C
    5. s : A
    6. t : B × C
    ⊢ C

The first subgoal is pretty easy, `assumption` should handle that. In
the other two we want to eliminate 6 and *then* we should be able to
apply assumption. In order to deal with this we use `|` to encode that
disjunction. In particular we want to run `assumption` OR `elim #6;
assumption` leaving us with

``` jonprl
    {
      *{unfold <prod>}; auto; elim #4; (assumption | elim #6; assumption)
    }
```

This completes the proof!

``` jonprl
    Theorem assoc :
      [Π(U{i}; A.
       Π(U{i}; B.
       Π(U{i}; C.
       Π(prod(A; prod(B;C)); _.
       prod(prod(A;B); C)))))] {
      *{unfold <prod>}; auto; elim #4; (assumption | elim #6; assumption)
    }.
```

As an exercise to the reader, prove that we can associate the other way.

## What on earth did we just do!?

So we just proved a theorem.. but what really just happened? I mean
how did we go from "Here we have an untyped computation system which
types just behaving as normal terms" to "Now apply `auto` and we're
done!". In this section I'd like to briefly sketch the path from
untyped computation to theorems.

The path looks like this

  - We start with our untyped language and its notion of computation

    We already discussed this in great depth before.

  - We define a judgment `a = b ∈ A`.

    This is a judgment, not a term in that language. It exists in
    whatever metalanguage we're using. This judgment is defined across
    3 terms in our untyped language (I'm only capitalizing `A` out of
    convention). This is supposed to represent that `a` and `b` are
    equal elements of type `A`. This also gives meaning to typehood:
    something is a type in CTT precisely when we know what the partial
    equivalence relation defined by `- = - ∈ A` on canonical values
    is.

    Notice here that I said *partial*. It isn't the case that `a = b ∈
    A` presupposes that we know that `a : A` and `b : A` because we
    don't have a notion of `:` yet!

    In some sense this is where we depart from a type theory like Coq
    or Agda's. We have programs already and on top of them we define
    this 3 part judgment which interacts which computation in a few
    ways I'm not specifying. In Coq, we would specify *one* notion of
    equality, generic over all types, and separately specify a typing
    relation.

 - From here we can define the normal judgments of Martin Lof's type
   theory. For example, `a : A` is `a = a ∈ A`. We recover the
   judgment `A type` with `A = A ∈ U` (where `U` here is a universe).

Hypothetical judgments are introduced in the same way they would be in
Martin-Lof's presentations of type theory. The idea being that `H ≫ J`
if `J` is evident under the assumption that each term in `H` has the
appropriate type and furthermore that `J` is functional (respects
equality) with respect to what `H` contains. This isn't really a
higher order judgment, but it will be defined in terms of a higher
order hypothetical judgment in the metatheory.

With this we have something that walks and quacks like normal type
theory. Using the normal tools of our metatheory we can formulate
proofs of `a : A` and do normal type theory things. This whole
development is building up what is called "Computational Type
Theory". The way this diverges from Martin-Lof's extensional type
theory is subtle but it does directly descend from Martin-Lof's famous
1979 paper "Constructive Mathematics and Computer Programming" (which
you should read. Instead of my crappy blog post).

Now there's one final layer we have to consider, the PRL bit of
JonPRL. We define a new judgment, `H ⊢ A [ext a]`. This is judgment is
cleverly set up so two properties hold

 - `H ⊢ A [ext a]` should entail that `H ⊢ a : A` or `H ⊢ a = a ∈ A`
 - In `H ⊢ A [ext a]`, `a` is an output and `H` and `A` are inputs. In
   particular, this implies that in any inference for this judgment,
   the subgoals may not use `a` in their `H` and `A`.

This means that `a` is completely determined by `H` and `A` which
justifies my use of the term output. I mean this in the sense of Twelf
and logic programming if that's a more familiar phrasing (for the
dozen Twelf users in the world). It's this judgment that we see in
JonPRL! Since that `a` is output we simply hide it, leaving us with `H
⊢ A` as we saw before. When we prove something with tactics in JonPRL
we're generating a *derivation*, a tree of inference rules which
make `H ⊢ A` evident for our particular `H` and `A`! These rules
aren't really programs though, they don't correspond one to one with
proof terms we may run like they would in Coq. The computational
interpretation of our program is bundled up in that `a`.

To see what I mean here we need a little bit more
machinery. Specifically, let's look at the rules for the equality
around the proposition `=(a; b; A)`. Remember that we have a term `<>`
lying around,

         a = b ∈ A
    ————————————————————
    <> = <> ∈ =(a; b; A)

So the only member of `=(a; b; A)` is `<>` `a = b ∈ A` actually
holds. First off, notice that `<> ∈ A` and `<> ∈ B` doesn't imply
that `A = B`! In another example, `λ(x. x) ∈ Π(A; _.A)` for all `A`!
This is a natural consequence of separating our typing judgment from
our programming language. Secondly, there's not really any computation
in the `e` of `H ⊢ =(a; b; A) (e)`. After all, in the end the only
thing `e` could be so that `e : =(a; b; A)` is `<>`! However, there
is potentially quite a large derivation involved in making `=(a; b;
A)` evident! For example, we might have something like this

    x : =(A; B; U{i}); y : =(b; a; A) ⊢ =(a; b; B)
    ———————————————————————————————————————————————— Substitution
    x : =(A; B; U{i}); y : =(b; a; A) ⊢ =(a; b; A)
    ———————————————————————————————————————————————— Symmetry
    x : =(A; B; U{i}); y : =(b; a; A) ⊢ =(b; a; A)
    ———————————————————————————————————————————————— Assumption

Now we write derivations of this sequent up side down, so the thing we
want to show starts on top and we write each rule application and
subgoal below it (AI people apparently like this?). Now this was quite
a derivation, but if we fill in the missing `[ext e]` for this derivation
from the bottom up we get this


    x : =(A; B; U{i}); y : =(b; a; A) ⊢ =(a; b; B)
    ———————————————————————————————————————————————— Substitution [ext <>]
    x : =(A; B; U{i}); y : =(b; a; A) ⊢ =(a; b; A)
    ———————————————————————————————————————————————— Symmetry     [ext <>]
    x : =(A; B; U{i}); y : =(b; a; A) ⊢ =(b; a; A)
    ———————————————————————————————————————————————— Assumption   [ext x]

Notice how at the bottom there was some computational content (That
`x` signifies that we're accessing a variable in our context) but than
we throw it away right on the next line! That's because we find that
no matter what the extract was that let's us derive `=(b; a; A)`, the
only realizer it could possible generate is `<>`. Remember our
conditions, if we can make evident the fact that `b = a ∈ A` then `<>
∈ =(b; a; A)`. Because we somehow managed to prove that `b = a ∈ A`
holds, we're entitled to just use `<>` to realize our proof. This
means that despite our somewhat tedious derivation and the
bookkeeping that we had to do to generate that program, that program
reflects *none* of it.

This is why type checking in JonPRL is woefully undecidable in part,
the realizers that we want to type check contain none of the helpful
hints that proof terms in Coq would. This also means that extraction
from JonPRL proofs is built right into the system and we can actually
generate cool and useful things! In NuPRL they actually write proofs
and use this realizers to run real software. From what they said at
OPLSS they can actually get these programs to run fast (within 5x of
naive C code).

So to recap, in JonPRL we

 - See `H ⊢ A`
 - Use tactics to generate a derivation of this judgment
 - Once this derivation is generated, we can extract the
 computational content as a program in our untyped system

In fact, we can see all of this happen if you call JonPRL from the
command line *or* hit C-c C-c in emacs! On our earlier proof we see


    Operator prod : (0; 0).
    ⸤prod(A; B)⸥ ≝ ⸤A × B⸥.

    Theorem left-id1 : ⸤⊢ ΠA ∈ U{i}. (prod(unit; A)) => A⸥ {
      fun-intro(A.fun-intro(_.prod-elim(_; _.t.t); prod⁼(unit⁼; _.hyp⁼(A))); U⁼{i})
    } ext {
      λ_. λ_. spread(_; _.t.t)
    }.

    Theorem left-id2 : ⸤⊢ ΠA ∈ U{i}. (prod(A; unit)) => A⸥ {
      fun-intro(A.fun-intro(_.prod-elim(_; s._.s); prod⁼(hyp⁼(A); _.unit⁼)); U⁼{i})
    } ext {
      λ_. λ_. spread(_; s._.s)
    }.

    Theorem assoc : ⸤⊢ ΠA ∈ U{i}. ΠB ∈ U{i}. ΠC ∈ U{i}. (prod(A; prod(B; C))) => prod(prod(A; B); C)⸥ {
      fun-intro(A.fun-intro(B.fun-intro(C.fun-intro(_.independent-prod-intro(independent-prod-intro(prod-elim(_;
      s.t.prod-elim(t; _._.s)); prod-elim(_; _.t.prod-elim(t;
      s'._.s'))); prod-elim(_; _.t.prod-elim(t; _.t'.t')));
      prod⁼(hyp⁼(A); _.prod⁼(hyp⁼(B); _.hyp⁼(C)))); U⁼{i}); U⁼{i});
      U⁼{i})
    } ext {
      λ_. λ_. λ_. λ_. ⟨⟨spread(_; s.t.spread(t; _._.s)), spread(_; _.t.spread(t; s'._.s'))⟩, spread(_; _.t.spread(t; _.t'.t'))⟩
    }.

Now we can see that those `Operator` and `≝` bits are really what we
typed with `=def=` and `Operator` in JonPRL, what's interesting here
are the theorems. There's two bits, the derivation and the extract
or realizer.

    {
      derivation of the sequent · ⊢ A
    } ext {
      the program in the untyped system extracted from our derivation
    }

We can move that derivation into a different theorem prover and check
it. This gives us all the information we need to prove that JonPRL is
safe and helps us not trust *all* of JonPRL (I wrote some of it so I'd
be a little scared to trust it :). We can also see the computational
bit of our proof in the extract. For example, the computation involved
in taking `A × unit → A` is just `λ_. λ_. spread(_; s._.s)`! This is
probably closer to what you've seen in Coq or Idris, even though I'd
say the derivation is probably more similar in spirit (just ugly and
beta normal). That's because the extract need not have any notion of
typing or proof, it's just the computation needed to produce a witness
of the appropriate type. This means for a really tricky proof of
equality, your extract might just be `<>`! Your derivation however
will always exactly reflect the complexity of your proof.

## Killer features

OK, so I've just dumped about 50 years worth of hard research in type
theory into your lap which is best left to ruminate for a
bit. However, before I finish up this post I wanted to do a little bit
of marketing so that you can see why one might be interested in JonPRL
(or NuPRL). Since we've embraced this idea of programs first and types
as PERs, we can define some really strange types completely
seamlessly. For example, in JonPRL there's a type `⋂(A; x.B)`, it
behaves a lot like `Π` but with one big difference, the definition of
`- = - ∈ ⋂(A; x.B)` looks like this

    a : A ⊢ e = e' ∈ [a/x]B
    ————————————————————————
       e = e' ∈ ⋂(A; x.B)

Notice here that `e` and `e'` *may not use* `a` anywhere in their
bodies. That is, they have to be in `[a/x]B` without knowing anything
about `a` and without even having access to it.

This is a pretty alien concept that turned out to be new in logic as
well (it's called "uniform quantification" I believe). It turns out
to be very useful in PRL's because it lets us declare things in our
theorems without having them propogate into our witness. For example,
we could have said

``` jonprl
    Theorem left-id2 :
          [⋂(U{i}; A.
           Π(prod(A; unit); _.
           A))] {
          unfold <prod>; auto; elim #2; assumption
        }.
```

With the observation that our realizer doesn't need to depend on `A`
at all (remember, no types!). Then the extract of this theorem is

    λx. spread(x; s._.s)

There's no spurious `λ _. ...` at the beginning! Even more wackily, we
can define subsets of an existing type since realizers need not have
unique types

    e = e' ∈ A  [e/x]P  [e'/x]P
    ————————————————————————————
      e = e' ∈ subset(A; x.P)

And in JonPRL we can now say things like "all odd numbers" by just
saying `subset(nat; n. ap(odd; n))`. In intensional type theories,
these types are hard to deal with and still the subject of open
research. In CTT they just kinda fall out because of how we thought
about types in the first place. Quotients are a similarly natural
conception (just define a new type with a stricter PER) but JonPRL
currently lacks them (though they shouldn't be hard to add..).

Finally, if you're looking for one last reason to dig into **PRL, the
fact that we've defined all our equalities extensionally means that
several very useful facts just fall right out of our theory

``` jonprl
    Theorem fun-ext :
      [⋂(U{i}; A.
       ⋂(Π(A; _.U{i}); B.
       ⋂(Π(A; a.ap(B;a)); f.
       ⋂(Π(A; a.ap(B;a)); g.

       ⋂(Π(A; a.=(ap(f; a); ap(g; a); ap(B; a))); _.
       =(f; g; Π(A; a.ap(B;a))))))))] {
      auto; ext; ?{elim #5 [a]}; auto
    }.
```

This means that two functions are equal in JonPRL if and only if they
map equal arguments to equal output. This is quite pleasant for
formalizing mathematics for example.

## Wrap Up

Whew, we went through a lot! I didn't intend for this to be a full
tour of JonPRL, just a taste of how things sort of hang together and
maybe enough to get you looking through the examples. Speaking of
which, JonPRL comes with quite a few [examples][examples] which are
going to make a lot more sense now.

Additionally, you may be interested in the documentation in the README
which covers most of the primitive operators in JonPRL. As for an
exhaustive list of tactics, [well...][tactic-issue].

Hopefully I'll be writing about JonPRL again soon. Until then, I hope
you've learned something cool :)


[jon]: http://www.jonmsterling.com/
[jonprl]: https://github.com/jonsterling/JonPRL
[nj]: http://www.smlnj.org/
[mode]: https://github.com/david-christiansen/jonprl-mode
[cpdt]: http://adam.chlipala.net/cpdt/
[examples]: https://github.com/jonsterling/JonPRL/tree/master/example
[tactic-issue]: https://github.com/jonsterling/JonPRL/issues/83
