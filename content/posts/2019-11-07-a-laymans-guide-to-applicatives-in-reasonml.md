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
a former colleague/mentor [Kris Nuttycombe](https://twitter.com/nuttycom):

> In functional programming, applicatives are the essence of parallel
processing, and monads are the essence of sequential processing.

In this post about applicative functors (aka applicatives), and my next
planned post about monads, I hope to dig into this notion, and try to impart
some intuition as to why this is true.

When I first learned about monads and applicatives, it took me longer to grok
applicatives, even though monads are an abstraction built on top of
applicatives. I was more used to the idea of monads from using things like
`flatMap` on lists and arrays (even though `flatMap` for list/array doesn't
really impart the feeling of sequential processing), and from using various
async types like Promises and Futures. Also, when writing basic application
code, you tend to do a lot of sequential processing (in both OO/imperative
and FP styles), which tend to relate more to monadic concepts - you often
have workflows where you compute a value, then use that value in the next
computation, and so on. Also, the `apply`/`ap`/`<*>` function (which we'll
soon see) is not something you tend to run into as much in its base form, so
it's not as immediately recognizable. However, you do see applicative
behavior when using things like JavaScript's `Promise.all` function, where
you pass in an array of `Promises` and get back and array of results,
assuming all the promises succeed, or a failed `Promise` if any of them fail.

The `apply`/`ap`/`<*>` function itself is just so strangely simple yet
mysterious, and to me, it was not immediately obvious how this humble
function lends itself to parallel processing.

# The plan

I'm going to approach the topic of applicatives in the way that I think would
have helped me to more quickly understand and appreciate it:

1. Introduce the `apply`/`ap`/`<*>` function and the `APPLY` typeclass to
show what it is, how simple it is, and to immediately dispel any notions of
magic
1. Introduce the `pure` function and the full-on `APPLICATIVE` typeclass
1. Mention why `apply` and `pure` are in separate typeclasses
1. Show some example implementations of `APPLICATIVE`, to re-iterate how
simple and unmagical it is
1. Show how `apply` and `pure` relate to `FUNCTOR`'s `map`
1. Demonstrate how `map` and `apply`/`ap`/`<*>` functions can be
re-formulated into functions that makes the parallel capability more clear
(`tuple2`, `map2`, etc.)
    - Short but detailed walkthrough of how `apply` works
1. Introduce the **map/ap/ap** pattern, as I like to call it
1. Show the variation on the **map/ap/ap** pattern: **pure(f)/ap/ap**
1. Talk about applicative "effects"
1. Talk about applicative validation
1. Talk about `APPLY` extensions
1. Talk about the `APPLICATIVE` laws

# Applicative Programming with Effects paper

Before we jump in, I'll just put a link to the paper where I believe the
concept of applicative functors was first introduced in 2007. The paper is
called [Applicative Programming with
Effects](http://www.staff.city.ac.uk/~ross/papers/Applicative.html). As far
as academic papers go, it's actually very readable, so I'd recommend checking
it out at some point. That said, if you're new to FP, it's important to
remember that with academic papers you likely won't understand much or any of
it initially. If that's the case, don't worry about it - try to plant a few
seeds then come back to it in the future. If you're never are able to
understand it, that's okay too! You don't need to understand the full scope
of everything to make use of the parts you do understand. FP is not an
all-or-nothing paradigm.

# APPLY typeclass

The `APPLY` typeclass is an extension of the `FUNCTOR` typeclass that I
convered in my [blog post about
functors](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml). Being
an extension of another typeclass simply means that `APPLY` is everything
that `FUNCTOR` is, and must abide by the same laws as `FUNCTOR`, but it also
adds something new to the mix, along with some new laws. `APPLY` adds one new
function which is often called `apply`, `ap`, or as an operator `<*>`. I'll
try to use the name `apply` here, but might also refer to `ap` or `<*>`. I'll
cover the laws at the end of the article to avoid getting lost before we even
get started.

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

`map` applies a pure function to a value that's inside a functor context,
while `apply` can apply a function that's inside a functor context to a value
that's inside another functor context. This curiously simple function is what
unlocks the power of parallel processing, but if you don't see it yet, that's
okay, I didn't either!

# APPLICATIVE typeclass

I'm going to add one more small concept to the mix now, because it's quite
simple and it makes sense to just explain it together with `apply`. This new
concept is the `pure` function. All `pure` does it take a "pure value" of
type `'a`, and stick it in a functor context `t('a)`.

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

`pure` is a simple, but interesting function in that we now have the ability
to actually put a value into our functor context, whereas before, we could
only operate on values that were already in the context, using `map` and
`apply`.

# APPLY and APPLICATIVE historical notes

For some historical context from Haskell, I believe that when the applicative
functor was first identified as a distinct abstraction, the key parts had
already been sort of "identified" as functions or concepts, but the typeclass
itself hadn't been officially split off from `Monad` yet. Some work has since
been done to split out an `Applicative` typeclass, which includes `pure` and
`<*>`. Languages/FP libraries that are newer and don't have the burden of
maintaining backwards compatibility seem to be separating the concept of
`APPLY` and `APPLICATIVE`, the reason being that there are things that can
conform to `APPLY`, but not `APPLICATIVE`, so it makes sense to keep those
abstractions separate.

For the rest of the article, we're going to just focus on `APPLICATIVE` as a
whole because it's useful to have both `apply` and `pure` at our disposal.
However, this is a good time to mention that when faced with a problem, you
should always try to follow the [rule of least
power](https://en.wikipedia.org/wiki/Rule_of_least_power). Use the
abstraction that does what you need with the least amount of power - this
makes your code more abstract, which means there are fewer ways for it to do
the wrong thing or be used incorrectly, and makes it more general so that
more things can use it, or be used with it.

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

In fact, the `Option` module itself actually conforms to `APPLICATIVE`
already because it has all the necessary parts, so it's not even strictly
necessary to define `Functor` and `Applicative`, but I like to do it anyway
to be explicit, and it feels more like Haskell/PureScript typeclass
instances. Also, I like to annotate the module types, just to make it clear
what my intentions are with these modules, even though the compiler can infer
the module types.

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
value, and then resolve with the function applied to the value.

One quick observation here is that in theory, we should be able to fire off
both of these promises at the same time, as they are not dependent on
one-another - the promise of `'a => 'b` doesn't care about the promise of
`'a` and vice versa - neither of them need to wait for the other to do its
job. Our `apply` function has to wait for both of these promises, because we
need the function and the value at the same time to use them together, but
the input promises themselves are independent.

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

We're using `Js.Promise.then_` here to just wait for the promise of the `'a
=> 'b` function to resolve, then we wait for the promise of the `'a` value to
resolve, and then we finally resolve the chain with `aToB(a)` to get our `'b`
value. This chain might look like we're making this operation sequential, but
remember that both promises are already running when we get them so the inner
promise might finish before the outer, but it doesn't matter to us. We need
them both to resolve before we can get our `'b` value. If we were
**constructing** the inner promise here, it would matter, but the promise was
already constructed and running when we got it.

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

As for `pure`, we just need to take a value of type `'a` and get it into the
`Js.Promise` context, so we can use `Js.Promise.resolve`.

Here is the full `Promise` module:

```ocaml
module Promise = {
  type t('a) = Js.Promise.t('a);

  let map = (aToB, promiseA) => {
    promiseA |> Js.Promise.then_(a => Js.Promise.resolve(aToB(a)));
  };

  let apply = (promiseAToB: Js.Promise.t('a => 'b), promiseA: Js.Promise.t('a)) => {
    promiseAToB
    |> Js.Promise.then_(aToB =>
        promiseA |> Js.Promise.then_(a => Js.Promise.resolve(aToB(a)))
      );
  };

  let pure = a => Js.Promise.resolve(a);

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

The implementation of `apply` for `Js.Promise` gives us a first glimpse as to
why applicatives are associated with parallel processing - we get two
independent "effectful values" (the promises), and we wait for both of them
to finish independently before we use them to do our final computation. I'll
talk about what "effectful values" means a little later.

# List/array/tree applicative

The types `list('a)`, `array('a)`, and other "multi-value data types" like
binary trees, etc. have applicative instances, but I'm going to skip these in
this article, as I don't personally think they are immediately helpful in
gaining the initial intuition about applicatives.

If you think about the signature of `apply` for a `list('a)`, you'd get the
following:

```ocaml
let apply = (list('a => 'b), list('a)) => list('b);
```

Basically, you have a list of functions and a list of values, and you need to
apply some or all of the functions to some or all the values. You can
probably imagine how you might end up with a cartesian product where all the
functions are applied to all the values to produce a new longer list of
un-obvious utility. The same idea applies for `array('a)`. For a binary tree,
imagine a binary tree of functions `Tree.t('a => 'b)` applied to a binary
tree of values `Tree.t('a)`. These applicatives can be implemented, but it's
not something you run into as much in day-to-day use.

# Result applicative

Now let's try to implement `APPLICATIVE` for `Result.t('a, 'e)`. Since
`FUNCTOR`/`APPLY`/`APPLICATIVE` want to operate on a type `t('a)`, we'll use
the module functor trick (see the [functors
article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml.md) again
to "lock-in" our error type:

```ocaml
module type TYPE = {
  type t;
};

module type FUNCTOR = {
  type t('a);
  let map: ('a => 'b, t('a)) => t('b);
};

module type APPLY = {
  include FUNCTOR;
  let apply: (t('a => 'b), t('a)) => t('b);
};

module type APPLICATIVE = {
  include APPLY;
  let pure: 'a => t('a);
};

module Result = {
  type t('a, 'e) = | Ok('a) | Error('e);

  let map = (aToB, resultA) => switch(resultA) {
    | Ok(a) => Ok(aToB(a))
    | Error(e) => Error(e)
  };

  let apply = (resultAToB, resultA) => switch((resultAToB, resultA)) {
    | (Ok(aToB), Ok(a)) => Ok(aToB(a))
    | (Error(e1), Ok(a)) => Error(e1)
    | (Ok(aToB), Error(e2)) => Error(e2)
    | (Error(e1), Error(e2)) => Error(e1) // !!!
  };

  let pure = a => Ok(a);

  module WithError = (E: TYPE) => {
    module Functor: FUNCTOR with type t('a) = t('a, E.t) = {
      type nonrec t('a) = t('a, E.t);
      let map = map;
    };

    module Apply: APPLY with type t('a) = t('a, E.t) = {
      include Functor;
      let apply = apply;
    };

    module Applicative: APPLICATIVE with type t('a) = t('a, E.t) = {
      include Apply;
      let pure = pure;
    };
  };
};
```

The implementation of `pure` is obvious because there's only one way to turn
a value of type `'a` into a `Result.t('a, 'e)`; however, the implementation
of `apply` poses an interesting conundrum in the `| (Error(e1), Error(e2))`
case - we have two errors and we have no idea what they are, but we need to
return a single error value in the resulting `Error('e)` constructor. We
don't know what the `'e` value is, so we don't have enough information to
know how to pick one error or the other, or combine them, so we'll just pick
one side (`e1` in this case), and fail the function. The only way we can
succeed is in the `| (Ok(aToB), Ok(a))` case - we need both the function and
the value in order to produce the result of type `'b` with `Ok(aToB(a))`.

We'll leave `Result` like this for now, but we'll revisit this error handling
issue later in the section on "applicative validation".

# Function applicative

As we saw in the [functor
article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml.md), we
could implement `FUNCTOR` for the function of type `'x => 'a`. Let's see how
to implement `APPLICATIVE` for this type:

```ocaml
module Function = {
  type t('x, 'a) = 'x => 'a;

  let map = (aToB: 'a => 'b, xToA: 'x => 'a) => {
    x => aToB(xToA(x));
  };

  let apply = (xToAToB: 'x => ('a => 'b), xToA: 'x => 'a) => {
    x => {
      let aToB = xToAToB(x);
      let a = xToA(x);
      aToB(a);
    };
  };

  let pure = (a: 'a): ('x => 'a) => {
    _ => a;
  };

  module WithArgument = (X: TYPE) => {
    module Functor: FUNCTOR with type t('a) = t(X.t, 'a) = {
      type nonrec t('a) = t(X.t, 'a);
      let map = map;
    };

    module Apply: APPLY with type t('a) = t(X.t, 'a) = {
      include Functor;
      let apply = apply;
    };

    module Applicative: APPLICATIVE with type t('a) = t(X.t, 'a) = {
      include Apply;
      let pure = pure;
    };
  };
};
```

The `apply` function here is given a function from `'x => ('a => 'b)`, and a
function `'x => 'a`, and needs to return a function `'x => 'b`. So in this
resulting function, we are given an `'x`, and need to return a `'b`. To do
this, we use the `'x` to get our `'a => 'b` function from the first argument,
and the same `'x` to get our `'a` value from the second argument, and just
apply the function. This is a great exercise in "following the types" - we
don't know up-front what we're doing, but we just look at the types and work
it out. If you do this in FP, you'll often find that there's only one valid
way to actually implement something, like in this case.

The `pure` function is just given a value `'a` and needs to return a function `'x => 'a`,
but we already have our `'a`, so we just return a function that throws away the input
and returns our `'a`. This function is commonly called `const`:

```ocaml
let const = (a, _) => a;
```

This looks funny in ReasonML syntax, because it looks like a function that
takes an `a` argument and another ignored argument, and then just returns the
`a`, but if you think about it in terms of partial application, if you supply
the `a` argument, you now have a function `_ => a`. This is useful for
supplying a function that produces a constant value to something that wants
you to give it an `'a => 'b` function; hence the name `const`.

Overall, the usefulness of this `APPLICATIVE` instance for `'x => 'a` is not
immediately obvious, but we'll explore it more when we talk about the [reader
monad](https://google.com/search?q=reader%20monad).

Also, I wanted to show this as an example of an `APPLICATIVE` that's not just
a static data value, to demonstrate that these concepts can apply to
functions too.

# JSON decoder applicative

Let's show a more real-world example of an applicative: the JSON decoder
function we saw in the [functor article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml.md). I'll just jump right
into it, and explain after, but I encourage you to try it yourself too.

```ocaml
module Decoder = {
  module Error = {
    type t = | ExpectedBool(Js.Json.t) | ExpectedString(Js.Json.t) | Other;
  };

  type t('a) = | Decode(Js.Json.t => Result.t('a, Error.t));

  let map = (aToB, (Decode(jsonToResultA))) => {
    Decode(json => jsonToResultA(json) |> Result.map(aToB));
  };

  let apply = (Decode(jsonToResultAToB), Decode(jsonToResultA)) => {
    Decode(json => {
      let resultAToB: Result.t('a => 'b, Error.t) = jsonToResultAToB(json);
      let resultA: Result.t('a, Error.t) = jsonToResultA(json);
      Result.apply(resultAToB, resultA);
    })
  };

  let pure = a => Decode(_ => Result.pure(a));

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

The `apply` function here feels a lot like the `apply` function we saw for
`'x => 'a`, and that's because it is very similar: `Js.Json.t => Result.t('a,
Error.t))` is of a similar form to `'x => 'a`, except the `'a` value is just
buried in another applicative type `Result`! In `Decoder.apply` we have a
decoder to get our function, and a decoder to get our value, so we feed the
`Js.Json.t` value into each, but then we get a `Result.t('a => 'b, Error.t)`
and a `Result.t('a, Error.t)`. Here we have a function `'a => 'b` buried in a
`Result` and a value `'a` buried in a result - so we can just use
`Reuslt.apply` to get our `Reuslt.t('b, Error.t)`!

This example serves to show that you don't always need to completely
understand what you're doing to create the typeclass instances for a type, as
long as you can do it, and your implementation follows the laws. Sometimes
you just need to follow the types! The concept of a "decoder of a function
`'a => 'b`" doesn't make much intuitive sense, but we'll soon see where this
comes into play, and maybe it will become clear as to why we'd do these strange
things.

# map in terms of apply and pure

We've now seen that `APPLICATIVE` is an extension of `FUNCTOR` - it' adds the
functions `apply` and `pure` to the `t('a)` and `map` function we got from
`FUNCTOR`. One interesting thing to note at this point is that you can actually
implement `map` in terms of just `apply` and `pure`:

```ocaml
// (pseudocode)

let apply: (t('a => 'b), t('a)) => t('b) = ...;

let pure = 'a => t('a) = ...;

let map = (aToB, fa) => apply(pure(aToB), fa);
```

Here, our `map` function is given a function `'a => 'b` and a value of type
`t('a)`. We "lift" the function into our applicative context `t` using
`pure(aToB)`, which gives us a value of type `t('a => 'b)`, then we apply
that "wrapped" or "effectful" function to our `t('a)` value using `apply`,
because that's what `apply` does:

```ocaml
let apply: (t('a => 'b), t('a)) => t('b);
```

When we get into monads, we'll also see that you can implement `apply` in
terms of the monadic function `flatMap` (aka `bind` or `>>=`).

I just wanted to point this out, as sometimes it's actually easier or more
convenient to implement one of the "more powerful" functions for a type, and
then just implement the "lesser" functions in terms of the more powerful
ones. I don't understand all or really any of the underlying math for this,
but I find it fascinating!

# Reformulation of apply

Let's now look at a reformulation of `apply` that better demonstrates the
parallel nature of applicatives. If this section bends your brain too far,
I'd highly recommend just trying it for yourself - it took me awhile to get
this.

Our goal will be to come up with a function of the following type, using only
the powers offered to us by `APPLICATIVE` (`t('a)`, `map`, `apply`, and
`pure`), and nothing more:

```ocaml
let tuple2: (t('a), t('b)) => t(('a, 'b));
```

Given two "effectful" values `t('a)` and `t('b)`, let see if we can "run"
them both to get at the values `'a` and `'b`, then combine those value in a
tuple. We're going to assume the result is also wrapped in our applicative
context `t`, because given the types of `map`, `apply`, and `pure`, it
doesn't look like we have any way to "get rid of" or "get out of" our `t`
context - we can only put things into it using `pure`, and operate on things
inside of our context using `map` and `apply`.

Let's start with a function that can create our tuple:

```ocaml
let makeTuple2 = (a, b) => (a, b);
```

Since we're in a curried language, another way to think about this function is
like this:

```ocaml
let makeTuple2 = a => {
  b => (a, b);
};
```

These are both the same thing, but the second form might help with the
intuition - given an `'a`, we can return a function `'b => ('a, 'b)`. The
first `makeTuple2` can also do this just as well, but it's not as clear with
the ReasonML syntax of `(a, b) => (a, b)` - this looks like a function of
arity 2 that just returns a tuple, but in reality, it's a curried function in
ReasonML - a function of a single argument that returns another function of a
single argument.

So back to our `tuple2` challenge:

```ocaml
let tuple2: (t('a), t('b)) => t(('a, 'b));
```

We're given a `t('a)` and a `t('b)`, and we have a function `makeTuple2: 'a
=> ('b => ('a, 'b))`. We have no way to get the `'a` value out of our
`t('a)`, we can only operate on the value inside the context using `map` and
`apply`. We have a value of the form `t('a)` (and a `t('b)`), but we don't
have a function of the form `t('a => 'b)`, so let's just try the only thing
that seems to make sense, `map: ('a => 'b, t('a)) => t('b)`. Let's just use
`option('a)` as our functor/applicative, just for simplicity.

Let's quickly consider our function `makeTuple2: 'a => ('b => ('a, 'b))` again.
If we write it like this, we can sort of fuzz the right hand side into an opaque
type like this:

```ocaml
'a => ('b => ('a, 'b));

'a => 'c

where 'c represents our 'b => ('a, 'b) function
```

Given this type `'c`, we can write our `map` like this:

```ocaml
let map: ('a => 'c, t('a)) => t('c);
```

and if we substitute `'c` with what it actually is:

```ocaml
let map: ('a => 'c,             , t('a)) => t('c            );
let map: ('a => ('b => ('a, 'b)), t('a)) => t('b => ('a, 'b));
```

So if we do this:

```ocaml
let fa: t('a) = ...;
let fBToAB = map(makeTuple2, fa);
```

Our `fBToAB` has the type `t('b => ('a, 'b))`

Let's use options to make this more concrete:

```ocaml
let makeTuple2 = (a, b) => (a, b);

let optionA = Some(42);

let optionB = Some("hi");

// Map the pure function on our optionA
// This results in a function 'b => ('a, 'b) **that is inside an option**

let optionBToResult: option('b => (int, 'b)) = Option.map(makeTuple2, optionA);

// Now we have a function inside an option, and a value `b inside
// an option, and that's what apply deals with:

let result: option((int, string)) = Option.apply(optionBToResult, optionB);

// result == Some((42, "hi"))
```

The above code bent my mind for awhile before I got it. If you're not getting
it just by looking, try working through it it yourself, and look carefully at
the types along the way. Basically, we're using `map` to apply the pure
function to our `option('a)` for the first step, then using `apply` to apply
a function (which is now inside an `option` from the `map`) to an
`option('b)`.

The even more surprising thing is that this pattern of `map` and `apply` works
for any number of arguments:

```ocaml
let makeTuple3 = (a, b, c) => (a, b, c);

let optionA = Some(42);
let optionB = Some("hi");
let optionC = Some(true);

let optionBToCToTuple = Option.map(makeTuple3, optionA);
let optionCToTuple = Option.apply(optionBToCToTuple, optionB);
let optionTuple = Option.apply(optionCToTuple, optionC);

// optionTuple = Some((42, "hi", true))
```

This works for any number of arguments. You can think of each step of `map`
and `apply` "filling" one of the arguments of hte initial function, but it's
all done inside of our applicative context. Put differently, each `apply`
step is given a function inside our applicative context, and applies it to a
value in an applicative context, to produce a function in an applicative
context. Once all the arguments are filled, we are left with the result - in
our case, the tuple. If any of the input values is `None`, think about how
`apply` is implemented for `option` - if either side `t('a => 'b)` or `t('a)`
is `None`, the result is `None`, and the remaining calls will just be `None`,
so the result will be `None`. The same phenomenon occurs with `Result.t('a,
'e)` - if any of the computations fail, the entire thing fails. Later on,
we'll see an interesting alternative way of dealing with errors using
applicative validation and the `Validation.t('a, 'e)` type (which we haven't
seen before).

In the end, we can define our `tuple2` function like this, assuming `A` here
is a type that has an `Applicative` instance (which is not easily expressed
in ReasonML without using module functors):

```ocaml
// This is kind of pseudo-code...

let tuple2 = (fa: A.t('a), fb: A.t('b)): A.t(('a, 'b)) => {
  A.apply(A.map(makeTuple2, fa), fb);
};
```

Basically the idea is we use `map` to apply our pure function `('a, 'b) =>
('a, 'b)` to the first wrapped/effectful value `A.t('a)`, which produces a
wrapped function. Then we use `apply` to apply that wrapped function to our
other wrapped value `A.t('b)`.

This `tuple2: (t('a), t('b)) => t(('a, 'b))` function is sometimes called
`product`, `product2` or `zip`, because it takes two effectful values and
combines them into the most basic product type - a tuple. I hope to write
about product and sum types in a later post.

If you got lost somewhere along the way, I'd again recommend trying it
yourself with a concrete type like `option`. It's hard to explain, but once
you slog through it enough times, you'll get it. Try implementing these
functions too, to see how the patterns expands to more values:

```ocaml
let tuple3: (t('a), t('b), t('c)) => t(('a, 'b, 'c));
let tuple4: (t('a), t('b), t('c), t('d)) => t(('a, 'b, 'c, 'd));
// etc.
```

Hint: you can implement `tuple3` in terms of some combination of `tuple2`,
`map`, `apply`, and/or `pure`. `tuple4` can be implemented in terms of
`tuple3` and friends etc.


# The map/ap/ap pattern

Let's get super fancy, and cast off our fear of weird operators, and do the
above using some Haskell-style infix operators. Once you grok the pattern,
the operator-based appraoch actually become quite beautiful. The usual caveat
with operators applies - they add a level of opacity and abstraction that is
quite hostile to newcomers, so it's best to introduce them with some
hand-holding, and not just dump them on people without the pre-requisite
setup.