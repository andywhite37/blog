--- 
date: 2019-11-07T22:28:54-07:00
title: "A Layman's Guide to Applicatives in ReasonML"
#series: ["A Layman's Guide to Functional Programming"]
categories: ["Software Development", "Functional Programming"]
tags: ["ReasonML", "OCaml", "Applicatives", "Functors", "Layman's Guide"]
draft: true
---

# Other posts in this series:

- [A Layman's Guide to Functors in ReasonML](2019-11-01-a-laymans-guide-to-functors-in-reasonml)

# Applicative functors

I'll start this post off with a tantalizing quote that I first heard from 
a former collegue/mentor [Kris Nuttycombe](https://twitter.com/nuttycom):

> In functional programming, applicatives are the essence of parallel
processing, and monads are the essence of sequential processing.

In this post about applicative functors (aka applicatives), and my next
planned post about monads, I hope to dig into this notion, and try to impart
some intuition as to why this is true.

When I first learned about monads and applicatives, it took me longer to grok
applicatives than monads, even though monads are an abstraction built on top
of applicatives. I was more used to the idea of monads from using things like
`flatMap` on lists and arrays (even though `flatMap` for list/array doesn't
really impart the feeling of sequential processing), and from using various
async types like promises. Also, when writing basic application code, you
tend to do a lot of sequential processing (in both OO/imperative and FP
styles), which tend to relate more to monadic concepts - you often have
workflows where you compute a value, then use that value in the next
computation, and so on. Also, the `apply`/`ap`/`<*>` function (which we'll
soon see) just seems so strangely simple yet mysterious, and to me, it was
not immediately obvious how this humble function allows for parallel
processing.

# The plan

I'm going to approach this topic in the way that I think would have helped me to
more quickly understand and appreciate it:

1. Introduce the `apply`/`ap`/`<*>` function and the `APPLY` typeclass to
show what it is, how simple it is, and to immediately dispel any notions of
magic
1. Introduce the `pure` function and the full-on `APPLICATIVE` typeclass
1. Mention why `apply` and `pure` in separate typeclasses, even though they
are good friends
1. Show some example implementations of `APPLICATIVE`, to re-iterate how simple and unmagical it is
1. Show how `apply` and `pure` relate to `FUNCTOR`'s `map`
1. Short but detailed walkthrough how `apply` actually does its thing
1. Demonstrate how `map` and `apply`/`ap`/`<*>` functions can be
re-formulated into functions that makes the parallel capability more clear
(`tuple2`, `map2`, etc.)
1. Introduce the **map/ap/ap/ap** pattern, as I like to call it
1. Show the variation on the **map/ap/ap/ap** pattern: **pure(f)/ap/ap/ap**
1. Talk about applicative "effects"
1. Talk about applicative validation
1. Talk about `APPLY` extensions
1. Talk about the `APPLICATIVE` laws

# Applicative Programming with Effects paper

Before we jump in, I'll just put a link to the paper where I believe the
distinct concept of applicative functors was first introduced in 2007. The
paper is called [Applicative Programming with
Effects](http://www.staff.city.ac.uk/~ross/papers/Applicative.html). As far
as academic papers go, it's actually very readable, so I'd recommend checking
it out at some point. With academic papers, if you're new to FP, it's
important to remember that you likely won't understand much or any of it. If
that's the case, don't worry about it - just keep digging, and try coming
back to it in the future. If you never are able to understand it, that's okay
too! You don't need to understand the full scope of everything to make use of
the parts you do understand.

# APPLY typeclass

The `APPLY` typeclass is an extension of the `FUNCTOR` typeclass that I
convered in my [blog post about
functors](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml). Being
an extension of another typeclass simply means that `APPLY` is everything
that a `FUNCTOR` is, and must abide by the same laws as `FUNCTOR`, but it
adds something new to the mix, along with some new laws. `APPLY` adds one new
function to the mix, which is often called `apply`, `ap`, or as an operator
`<*>`. I'll try to use the name `apply` here, but might also refer to `ap` or
`<*>`. I'll cover the laws at the end of the article to avoid getting lost
before we even get going.

We can define the `APPLY` typeclass as a ReasonML module type like this:

```ocaml
module type APPLY = {
  type t('a);

  let map   = (  'a => 'b,  t('a)) => t('b);

  let apply = (t('a => 'b), t('a)) => t('b);
};
```

As you can see, `APPLY` has a `type t('a)`, the `map` function we know from
`FUNCTOR`, and the new function `apply`. The `apply` function is the only
difference compared to `FUNCTOR`.

Using the module `include` mechanism
from OCaml/ReasonML, we can define `APPLY` like this too:

```ocaml
module type FUNCTOR = {
  type t('a);

  let map = ('a => 'b, t('a)) => t('b);
};

module type APPLY = {
  include FUNCTOR;

  let apply = (t('a => 'b), t('a)) => t('b);
};
```

The `include` here more clearly illustrates the relationship between
`FUNCTOR` and `APPLY`. If you haven't seen `include`, it's basically like a
module-level copy or spread - it takes whatever's inside the module you're
`including` and spreads it into the module that has the `include`. So in this
case, we're including the `type t('a)` and `let map = ...` from `FUNCTOR`,
and then we just add our `apply` after that.

If you looked closely at the first example, you
might have noticed that I aligned the `'a` and `'b` parameters to
illustrate a similarity between `map` and `apply`:

```ocaml
let map   = (  'a => 'b,  t('a)) => t('b);
let apply = (t('a => 'b), t('a)) => t('b);
```

`map` applies a pure function to a value that's inside our functor context,
while `apply` can apply a function that's in our functor context to a value
that's in a separate functor context. This curiously simple function is what
unlocks the power of parallel processing, but if you don't see it yet, that's
okay, I didn't either!

# APPLICATIVE typeclass

I'm going to add one more small concept to the mix now, because it's quite
simple and it makes sense to just explain it together with `apply`. This new
concept is the `pure` function. All `pure` does it take a "pure value" of
type `'a`, and stick it in your functor context `t('a)`.

```ocaml
let pure: 'a => t('a);
```

Here, `'a` can be literally any value you want - the key point is that our
other functions just see it as `'a` - they don't know nor care what it is.

The `APPLICATIVE` typeclass is simply an extension of `APPLY` that adds this `pure`
function, and its corresponding laws:

```ocaml
module type APPLICATIVE = {
  type t('a);
  let map = ('a => 'b, t('a)) => t('b);
  let apply = (t('a => 'b), t('a)) => t('b);
  let pure = 'a => t('a);
};
```

Or we can write it using `include` like this:

```ocaml
module type FUNCTOR = {
  type t('a);
  let map = ('a => 'b, t('a)) => t('b);
};

module type APPLY = {
  include FUNCTOR;
  let apply = (t('a => 'b), t('a)) => t('b);
};

module type APPLICATIVE = {
  include APPLY;
  let pure = 'a => t('a);
};
```

`pure` is a simple, but interesting function in that we now have the power to
put a value into our functor context, whereas before, we could only operate
on values that were already in the context, using `map` and `apply`.

# APPLY and APPLICATIVE historical notes

This might not be correct, but In Haskell, I believe when applicatives were
first identified as a unique abstraction, the key parts `ap` (aka `apply`)
and `return` (aka `pure`) were already part of the `Monad` abstraction, and
were split off together into a new `Applicative` typeclass. Sometime later,
it was decided that `Apply` should be it's own abstraction and `Applicative`
should extend that, and I think that's the general direction of the FP community.

For the rest of the article, we're going to just focus on `APPLICATIVE` as a
whole because it's useful to have both `apply` and `pure` at our disposal.
However, this is a good time to mention that when faced with a problem, you
should always try to follow the [rule of least
power](https://en.wikipedia.org/wiki/Rule_of_least_power). Use the
abstraction that does what you need with the least amount of power - this
makes your code more abstract, which means there are fewer ways for it to do
the wrong thing, and makes it more general so that more things can use it.

Let's see what `APPLICATIVE` looks like in some real examples:

# Option applicative

Let's implement `APPLICATIVE` for `option('a)`. It's pretty easy - you just
follow the types. I'm going to implement all the functions at the top-level
of `Option` for convenience, then just alias the functions in the typeclass
instances. I'm also going to use `include` to deal with the hierarchy from `FUNCTOR`
up to `APPLICATIVE`. `include` works for both module types and modules, so we
can use it in both the typeclasses and the instances.

```ocaml
module Option = {
  type t(a') = option('a);

  let map = (aToB: 'a => 'b, optionA: option('a)) => switch(optionA) {
    | Some(a) => Some(aToB(a));
    | None => None;
  };

  let apply = (optionAToB: option('a => 'b), optionA: option('a)) =>
    switch(optionAToB, optionA) {
      | (Some(aToB), Some(a)) => Some(aToB(a))
      | (Some(aToB), None) => None
      | (None, Some(a)) => None
      | (None, None) => None
    };

  let pure = a => Some(a);

  module Functor: FUNCTOR with type t('a) = t('a) = {
    type nonrec t('a) = t('a);
    let map = map;
  };

  module Apply: APPLY with type t('a) = t('a) = {
    include Functor;
    let apply = apply;
  };

  module Applicative: APPLICATIVE with type t('a) = t('a) = {
    include Apply;
    let pure = pure;
  };
};
```

`apply` is pretty straightforward, because there aren't too many ways of
implementing it. You could of course just return `None` in all cases, but
that would violate the `APPLY` laws, which I'll cover at the end of the
article. You can only ever get a `'b` value if you have both the `'a => 'b`
function and the `'a` value, so the cases where we only have the function or
only the value, or neither just have to return `None`.

`pure` just takes a value and sticks it in `Some`.

Once I have `map`, `apply`, and `pure`, I can just create my instance modules
using these. As you can see, I define each typeclass separately, even though
`Applicative` can do everything a `Functor` can do. In ReasonML, this is not
strictly necessary to do, but I like to do it anyway, because it helps to
quickly identify what a particular type can do, and it serves as a constant
reminder of what each typeclass does - kind of like in-code, type-checked
documentation about a type.

That's it for `option`!

# Js.Promise applicative

Now let's try implementing `apply` for a more complex type: `Js.Promise.t('a)`.

First think about the type signature, and substitute `Js.Promise.t('a)` for `t('a)`:

```ocaml
let apply: (t('a => 'b), t('a)) => t('b);

let apply: (Js.Promise.t('a => 'b), Js.Promise.t('a)) => Js.Promise.t('b)
```

If you've worked with promises, it's probably not too hard to see how this is
implemented - you just need to wait for the `'a => 'b` function and the `'a`
value, and then resolve the promise with the function applied to the value.

One quick observation here is that in theory, we should be able to fire off
both of these promises at the same time, as they are not dependent on
one-another - the function promise doesn't care about the value promise and
vice versa - neither of them need the value from the other to do its job. Our
`apply` function has to wait for both of these things, because we need the
function and the value at the same time to use them together, but the input
promises themselves are independent.

That said, these promises have already been "fired off" before we even get
our hands on them in `apply`, because that's what JS promises do - they start
running as soon as you construct them! So in `apply` we get two
already-running hot promises, and we just need to wait for both of them to
finish.

```ocaml
module Promise = {
  let apply = (promiseAToB: Js.Promise.t('a => 'b), promiseA: Js.Promise.t('a)) => {
    promiseAToB
    |> Js.Promise.then_(aToB =>
        promiseA |> Js.Promise.then_(a => Js.Promise.resolve(aToB(a)))
      );
  };
};
```

We're using `Js.Promise.then_` here to just wait for the function to arrive,
then we wait for the value to arrive, and finally resolve the chain with
`aToB(a)` to get our `'b` value. This chain might look like we're making this
operation sequental, but remember that both promises are already running when
we get them so the inner promise might finish before the outer, but it
doesn't matter to us, because we need them both to resolve before we do the
thing. If we were **constructing** the inner promise here, it would matter,
but the promise was already constructed and running when we got it.

Example usage:

```ocaml
apply(Js.Promise.resolve(a => a * 2), Js.Promise.resolve(42))
|> Js.Promise.then_(a => Js.Promise.resolve(Js.log(a)));

// 84
```

We could have also "cheated" and just used `Js.Promise.all2` here, because
someone has already implemented for us:

```ocaml
let apply = (promiseAToB, promiseA) =>
  Js.Promise.all2((promiseAToB, promiseA))
  |> Js.Promise.then_(((aToB, a)) => Js.Promise.resolve(aToB(a)));
```

This isn't actually cheating - it's perfectly fine to use functions that
already exist to implement typeclasses, assuming they follow the laws.
Speaking of `all2`, we'll soon see how to implement this ourselves for **any
applicative**, not just `Js.Promise`, and we'll also see that by implementing
`apply` and `pure` for a type, we can get a ton of other stuff for free!