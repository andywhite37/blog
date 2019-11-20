--- 
date: 2019-11-07T22:28:54-07:00
title: "A Layman's Guide to Applicatives in ReasonML"
#series: ["A Layman's Guide to Functional Programming"]
categories: ["Software Development", "Functional Programming"]
tags: ["ReasonML", "OCaml", "Applicatives", "Functors", "Layman's Guide"]
draft: false
---

# Other posts in this series:

- [A Layman's Guide to Functors in ReasonML](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml)

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
computation, and so on. Additionally, the `apply`/`ap`/`<*>` function (which
we'll soon see) is not something you tend to run into as much in its base
form, so it's not as immediately recognizable. However, you do see
applicative behavior when using things like JavaScript's `Promise.all`
function, where you pass in an array of `Promises` and get back and array of
results, assuming all the promises succeed, or a failed `Promise` if any of
them fail.

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
1. Talk about `APPLY` extensions
1. Talk about applicative validation
1. Talk about the `APPLICATIVE` laws

# Applicative Programming with Effects paper

Before we jump in, I'll just put a link to the paper where I believe the
concept of applicative functors was first introduced in 2007. The paper is
called [Applicative Programming with
Effects](http://www.staff.city.ac.uk/~ross/papers/Applicative.html) by Conor
McBride and Ross Paterson. As far as academic papers go, it's very readable,
so I'd recommend checking it out at some point. That said, if you're new to
FP, it's important to remember that with academic papers you may not
understand much or any of it initially. If that's the case, don't worry about
it - try to plant a few seeds then come back to it in the future. If you're
never are able to understand it, that's okay too! You don't need to
understand the full scope of everything to make use of the parts you do
understand, and you don't need to fully grok all the underlying math and
academics. FP is not an all-or-nothing paradigm.

# APPLY typeclass

The `APPLY` typeclass is an extension of the `FUNCTOR` typeclass that I
covered in my [blog post about
functors](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml) Being an
extension of `FUNCTOR` simply means that `APPLY` is everything that `FUNCTOR`
is, and must abide by the same laws as `FUNCTOR`, but it also adds something
new to the mix, along with some new laws. `APPLY` adds one new function which
is often called `apply`, `ap`, or as an operator `<*>`. I'll try to use the
name `apply` here, but might also refer to `ap` or `<*>`. I'll cover the laws
at the end of the article to avoid getting lost before we even get started.

We can define the `APPLY` typeclass as a ReasonML module type like this:

```ocaml
module type APPLY = {
  type t('a);

  let map   : (  'a => 'b,  t('a)) => t('b);

  let apply : (t('a => 'b), t('a)) => t('b);
};
```

As you can see, `APPLY` has the `type t('a)` and the `map` function we know
from `FUNCTOR`, and adds the new function `apply`. The `apply` function is
the only difference compared to `FUNCTOR`.

Using the module `include` mechanism
from OCaml/ReasonML, we can define `APPLY` like this too:

```ocaml
module type FUNCTOR = {
  type t('a);

  let map: ('a => 'b, t('a)) => t('b);
};

module type APPLY = {
  include FUNCTOR;

  let apply: (t('a => 'b), t('a)) => t('b);
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
let map   : (  'a => 'b,  t('a)) => t('b);
let apply : (t('a => 'b), t('a)) => t('b);
```

`map` applies a **pure function** to a value that's inside a functor context,
while `apply` can apply a **function that's inside a functor context** to a
value that's inside another functor context. This curiously simple function
is what unlocks the power of parallel processing, but if you don't see it
yet, that's okay, I didn't either!

# APPLICATIVE typeclass

I'm going to introduce one more small concept now, because it's quite simple
and it makes sense to just explain it together with `apply`. This new concept
is the `pure` function. All `pure` does it take a **pure value** of type
`'a`, and stick it in a functor context `t('a)`. I say "pure value" to
differentiate it from an "effectful value," which is discussed later.

```ocaml
let pure: 'a => t('a);
```

Here, `'a` can be any type of value you want - the key point is that our
other functions just see it as an `'a` - they don't know nor care what it is.

The `APPLICATIVE` typeclass is simply an extension of `APPLY` that adds this `pure`
function, and its corresponding laws:

```ocaml
module type APPLICATIVE = {
  // From FUNCTOR
  type t('a);
  let map: ('a => 'b, t('a)) => t('b);

  // FROM APPLY
  let apply: (t('a => 'b), t('a)) => t('b);

  // For APPLICATIVE
  let pure: 'a => t('a);
};
```

Or we can write it using `include` like this:

```ocaml
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
```

`pure` is a simple, but interesting function in that we now have the ability
to actually put a value into our functor context, whereas before, we could
only operate on values that were already in the context, using `map` and
`apply`. Also note that we do not have a way to get a value out of our
applicative functor context once it's in there, i.e. we don't have a `t('a)
=> 'a` function here.

# APPLY and APPLICATIVE historical notes

For some historical context from Haskell, I believe that when the applicative
functor was first identified as a distinct abstraction, the key parts had
already been sort of "identified" as functions or concepts in the `Monad`
typeclass. Some work has been done to split off an `Applicative` typeclass,
which includes `pure` and `<*>` (the `apply` operator). Languages and FP
libraries that are newer and that don't have the burden of maintaining
backwards compatibility seem to be separating the concept of `APPLY` and
`APPLICATIVE` up front. The reason to split these is that there are things
that can conform to `APPLY`, but not `APPLICATIVE`, so it makes sense to keep
those abstractions separate.

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

Let's implement `APPLICATIVE` for `option('a)`. This one is pretty easy, you
just "follow the types." I'm going to implement all the functions at the
top-level of `Option` for convenience, then just alias the functions in the
typeclass instances. I'm also going to use `include` to deal with the
hierarchy from `FUNCTOR` up to `APPLICATIVE`. `include` works for both module
types and modules, so we can use it in both the typeclasses and the
instances.

```ocaml
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

As for `pure`, it just takes a value and sticks it in `Some`.

Once I have `map`, `apply`, and `pure`, I can just create my instance modules
`Functor`, `Apply`, and `Applicative` by aliasing the functions in the right
places. As you can see, I'm defining each typeclass separately, even though
`Applicative` can do everything `Apply` and `Functor` can do. In ReasonML,
this is not strictly necessary to do, but I like to do it anyway, because it
helps to quickly identify what a particular type can do, and it serves as a
constant reminder of what each typeclass does - kind of like in-code,
type-checked documentation about a type.

In fact, the `Option` module itself actually conforms to `APPLICATIVE`
already because it has all the necessary parts, so it's not even strictly
necessary to define `Functor`, `Apply` and `Applicative`, but I like to do it
anyway to be explicit, and it feels more like Haskell/PureScript typeclass
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
need the `'a => 'b` function and the `'a` value at the same time to use them
together to get the `'b`, but the input promises themselves are independent.

That said, these promises have already been "fired off" before we even get
our hands on them in `apply`, because that's what JS promises do - they start
running as soon as you construct them! So in `apply` we get two
already-running "hot" promises, and we just need to wait for both of them to
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

Another thing to note is that if either of the `Js.Promises` fail, the
`apply` function will fail. Again, we can only succeed if both inputs succeed
and give us our `'a => 'b` function and our `'a` value. With applicatives, we
actually have a few different options for handling errors, but here I'm going
to keep it simple and just fail fast using the default failure mechanism of
chained `Promises`.

Example usage:

```ocaml
Promise.apply(Js.Promise.resolve(a => a * 2), Js.Promise.resolve(42))
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
gaining an initial intuition about applicatives.

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
not something you run into as much in day-to-day use, so I'll not get into
these at this time, but you are welcome to give it a shot yourself!

# Result applicative

Now let's try to implement `APPLICATIVE` for `Result.t('a, 'e)`. Since
`FUNCTOR`/`APPLY`/`APPLICATIVE` want to operate on a type `t('a)`, we'll use
the module functor trick (see the [functors
article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml)) again
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
case - we have two errors (of the same type `'e`), but we need to return a
single error value in the resulting `Error('e)` constructor. We don't know
what the `'e` value is, so we don't have enough information to know how to
pick one error or the other, or combine them, so we'll just pick one side
(`e1` in this case), and fail the function. Just like with `option` and
`Js.Promise`, the only way we can succeed is in the `| (Ok(aToB), Ok(a))`
case - we need both the function and the value in order to produce the result
of type `'b` with `Ok(aToB(a))`.

We'll leave `Result` like this for now, but we'll revisit this error handling
issue later in the section on "applicative validation".

# Function applicative

As we saw in the [functor
article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml), we
could implement `FUNCTOR` for the function of type `'x => 'a`. Let's see how
to implement `APPLICATIVE` for this type:

```ocaml
module Function = {
  type t('x, 'a) = 'x => 'a;

  let map = (aToB: 'a => 'b, xToA: 'x => 'a): ('x => 'b) => {
    x => aToB(xToA(x));
  };

  let apply = (xToAToB: 'x => ('a => 'b), xToA: 'x => 'a): ('x => 'b) => {
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

The `apply` function here is given a function `'x => ('a => 'b)`, and a
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
creating a function that just produces a constant value regardless of the
input, hence the name `const`. It's another one of those utilities like
`identity: 'a => 'a` that is so trivially simple, but sometimes it's exactly
what you need.

Overall, the usefulness of this `APPLICATIVE` instance for `'x => 'a` is not
immediately obvious, but we'll explore it more when we talk about the [reader
monad](https://google.com/search?q=reader%20monad).

Also, I wanted to show this as an example of an `APPLICATIVE` that's not just
a static data value, to demonstrate that these concepts can apply to
functions too.

# JSON decoder applicative

Let's show a more real-world example of an applicative: the JSON decoder
function we saw in the [functor article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml). I'll just jump right
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
buried in another (applicative) type `Result`. In `Decoder.apply` we have a
**decoder of a function**, and a **decoder of a value**. Recall that a decoder
is just a function that takes a `Js.Json.t` value and produces a `Result`, so
we feed the input `Js.Json.t` value into each decoder argument to get a
`Result.t('a => 'b, Error.t)` and a `Result.t('a, Error.t)`. Here we have a
function `'a => 'b` buried in a `Result` and a value `'a` buried in a
`Result` - so we can take advantage of the fact that `Result` is also an
applicative functor, and just use `Result.apply` to apply the wrapped
function to our wrapped value to get our `Result.t('b, Error.t)`! Take note
that we've again not actually done any work at this point - no JSON has been
decoded - we've simply composed some functions to turn a decoder of `'a` into
a decoder of `'b`. Think about how you might do something like this in an
imperative or OO language - I'm pretty sure I would have done it in a much
less elegant way!

This example also serves to show that you don't always need to completely
understand what you're doing (or even the end result) to create the typeclass
instances for a type. As long as you can do it, the types line up, and your
implementation follows the laws, there's a good chance you did it correctly.
Sometimes all you need to do is just follow the types! We're kind of glossing
over the laws, and we haven't yet seen how to test the laws, but we'll
hopefully try that out in another blog post. The concept of a "decoder of a
function" `'a => 'b` doesn't make much intuitive sense, but we'll soon see
where this comes into play, and maybe it will become clear as to why we're
doing these things.

# map in terms of apply and pure

We've now seen that `APPLICATIVE` is an extension of `FUNCTOR` - it adds the
functions `apply` and `pure` to the `t('a)` and `map` function we have from
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
that "wrapped" function to our `t('a)` value using `apply`, because that's
what `apply` does:

```ocaml
let apply: (t('a => 'b), t('a)) => t('b);
```

When we get into monads, we'll also see that you can implement `apply` in
terms of the monadic function `flatMap` (aka `bind` or `>>=`).

I just wanted to point this out, as sometimes it's actually easier or more
convenient to implement one of the higher-level functions for a type, and
then just implement the lower-level functions in terms of the "more powerful"
ones.

# Reformulation of apply as tuple2

Let's now look at a reformulation of `apply` that better demonstrates the
parallel nature of applicatives. If this section bends your brain too far,
I'd highly recommend just trying it for yourself - it can take some time and
practice for this to sink in, especially if you're not used to working with
functions with curried arguments.

Our goal will be to come up with a function of the following type, using only
the powers offered to us by `APPLICATIVE`: `t('a)`, `map`, `apply`, and
`pure`, and nothing more. To be honest, we're actually just going to just use
`APPLY` here - we don't actually need `pure` for this.

```ocaml
let tuple2: (t('a), t('b)) => t(('a, 'b));
```

Given two "effectful" values `t('a)` and `t('b)`, let see if we can "run"
them both to get at the values `'a` and `'b`, then combine those values in a
"effectful" tuple. We're going to assume the result is also wrapped in our
applicative context `t`, because given the types of `map`, `apply`, and
`pure`, it doesn't look like we have any way to "get rid of" or "get out of"
our `t` context - we can only put things into it using `pure`, and operate on
things inside of our context using `map` and `apply`.

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
`apply`. We again have a value of the form `t('a)` (and a `t('b)`), but we
don't have a function of the form `t('a => 'b)` so `apply` doesn't seem
immediately applicable, but we do have our "pure" `makeTuple2` function, so
let's just try the only thing that seems to make sense, `map: ('a => 'b,
t('a)) => t('b)` 

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

If it's not clear, I'm just trying to demonstrate what happens when you map a
function of more than one argument over a functor whose type matches the type
of the first argument of our function - you end up with a **function inside
your functor context.**  And this function has one fewer argument than what
you started with.

Let's try mapping `makeTuple2` over our `fa`, which is a `t('a)`:

```ocaml
let fa: t('a) = ...;
let fBToAB: t('b => ('a, 'b)) = map(makeTuple2, fa);
```

Our `fBToAB` has the type `t('b => ('a, 'b))` - we mapped a function of
multiple arguments over our functor and ended up with a function of one fewer
arguments inside our functor context. This ability to `map` and `apply` a
function of multiple arguments over a bunch of individual functor values is
the key to how applicatives work.

After this `map` over the first value `t('a)`, we now have a function of the
form `t('x => 'y)` (actually `t('b => ('a, 'b))`). `map` can't deal with this
because `map` wants a pure/non-wrapped function, but `apply` knows what to do
with a wrapped function:

```ocaml
let fa: t('a) = ...;

let fb: t('b) = ...;

// map first - now we have a function inside a functor
let fBToAB: t('b => ('a, 'b)) = map(makeTuple2, fa);

// then apply this wrapped function to the wrapped 'b value
let fAB: t(('a, 'b)) = apply(fBToAB, fb);
```

We've now "filled" all the arguments of `makeTuple2`, and we get our final
result - a tuple wrapped in our applicative context: `t(('a, 'b))`.

To wrap up, we'll now write our `tuple2` function in terms of `map` and
`apply`. (Stay tuned to see a cleaner, more intuitive way to do this
below):

```ocaml
let tuple2 = (fa: t('a), fb: t('b)) => {
  let makeTuple2 = (a, b) => (a, b);
  apply(map(makeTuple2, fa), fb);
};
```

Let's run through a quick demonstration of this using options to make it more
concrete:

```ocaml
let makeTuple2 = (a, b) => (a, b);

let optionA = Some(42);

let optionB = Some("hi");

// Map the pure function on our optionA
// This results in a function 'b => ('a, 'b) **that is inside an option**

let optionBToTuple: option('b => (int, 'b)) = Option.map(makeTuple2, optionA);

// Now we have a function inside an option, and a value `b inside
// an option, and that's what apply deals with:

let optionTuple: option((int, string)) = Option.apply(optionBToTuple, optionB);

// optionTuple == Some((42, "hi"))
```

If you're not getting it just by looking, try working through it it yourself,
and look carefully at the types along the way. Basically, we're using `map`
to apply the pure function to our `option('a)` for the first step, then using
`apply` to apply a function (which is now inside an `option`) to our
`option('b)`.

One final note on this example: in the section where we implemented `map` in
terms of `apply` and `pure`, we saw that if we lifted a function into the
applicative context using `pure`, we could then just use `apply` to apply the
function to our effectful value. We could also implement `tuple2` in this
same style - we can lift the pure function into our context first, then just
`apply` it, then `apply` the resulting wrapped function to get the final
result:

```ocaml
let tuple2 = (fa: t('a), fb: t('b)) => {
  let makeTuple2 = (a, b) => (a, b);
  apply(apply(pure(makeTuple2), fa), fb);
};
```

# Apply multiple times

Surprisingly, this pattern of `map` and `apply` works for **any** number of
arguments - we can just keep `apply`ing the resulting functions to the next
values until we run out of arguments in our function and arrive at our final
result. That said, we will only succeed in getting the final result if each
step along the way is also successful. If any step fails, the whole
computation will fail, but we actually have some cool options for how we can
deal with errors.

If you think about how `apply` is implemented, it gets a wrapped function,
which basically carries information about the previous computations, and a
wrapped value, which is the "current value" on which we want to operate. The
previous computations may have already failed, but `apply` will still go
through the motion of considering each input value, regardless of what
happened in the past. That said, `apply` doesn't give us the ability to
"recover" a computation that failed - we can't create a successful result
unless we have both the `'a => 'b` function and the `'a` value, so all we can
do is propagate the previous errors, or possibly add our own error to the mix
(which we'll see later). This inability to fork or recover a computation is
one of the reasons that applicatives are strictly less powerful than monads.
However, being less powerful sometimes has its advantages - for example,
because an applicative can't fork the flow of a computation (i.e. recover
from an error, or create some new processing branch), we are forced to feed
it all the information we want to process at once. Knowing the full-scope of
the problem up-front can sometimes unlock certain optimizations and allow us
to make certain assumptions about how the computation will occur.

Anyway, to demonstrate how to apply the pattern to functions of more
arguments, below is an example of the `map`/`apply` pattern being used with a
(curried) arity 3 function:

```ocaml
let makeTuple3 = (a, b, c) => (a, b, c);

let optionA = Some(42);
let optionB = Some("hi");
let optionC = Some(true);

let optionBToCToTuple: option(('b, 'c) => (int, 'b, 'c)) = Option.map(makeTuple3, optionA);
let optionCToTuple: option('c => (int, string, 'c))  = Option.apply(optionBToCToTuple, optionB);
let optionTuple: option((int, string, bool)) = Option.apply(optionCToTuple, optionC);

// optionTuple = Some((42, "hi", true))
```

More succinctly, we could write it like this:

```ocaml
let tuple3 = (fa: t('a), fb: t('b), fc: t('c)) => {
  let f = (a, b, c) => (a, b, c);
  apply(apply(map(f, fa), fb), fc);
};

// or using pure to immediately lift our function into the applicative context:

let tuple3 = (fa: t('a), fb: t('b), fc: t('c)) => {
  let f = (a, b, c) => (a, b, c);
  apply(apply(apply(pure(f), fa), fb), fc);
};
```

Again, this pattern works for any number of arguments - just keep `apply`ing
until you "fill" all the arguments of your function. Try it yourself with
`tuple4` and so-on.

The `tuple2: (t('a), t('b)) => t(('a, 'b))` function is sometimes called
`product`, `product2` or `zip`, because it takes two effectful values and
combines them into the most basic product type - a tuple. I hope to write
about product and sum types in a later post. Looking at `tuple2` it's likely
more clear as to why applicatives are associated with parallel operations -
we start with two independent "effectful" values `t('a)`, and `t('b)`, and we
produce a value `t(('a, 'b))` that's a combination of our two inputs
(assuming they both "succeed").

Well, that was my attempt at describing how applicatives work, but not sure
how successful it was. If you got lost somewhere along the way, I'd again
recommend trying it yourself with a concrete type like `option`. It's hard to
explain, but once you slog through it enough times, you'll get it. Try
implementing these functions too, to see how the patterns expands to more
values:

```ocaml
let tuple4: (t('a), t('b), t('c), t('d)) => t(('a, 'b, 'c, 'd));

let tuple5: (t('a), t('b), t('c), t('d), t('e)) => t(('a, 'b, 'c, 'd, 'e));
// etc.
```

If you haven't recognized it yet, these look at lot like the
`Js.Promise.allN` functions, but there's no specific mention of `Js.Promise`
here - we're just using the functions from our `APPLY` abstraction.

# map2, map3, etc.

Now let's generalize our `tuple2`, `tuple3`, etc. functions. Recall the defintion
we made for `tuple2`:

```ocaml
let tuple2 = (fa: t('a), fb: t('b)) => {
  let makeTuple2 = (a, b) => (a, b);
  apply(map(makeTuple2, fa), fb);
};
```

The `makeTuple2` here is just a function that basically takes two inputs, and
"combines" them into something else. We can allow the caller to pass in this
function, so they can combine the values however they want, rather than
having to deal with a tuple. We'll call this function `map2`:

```ocaml
let map2 = (f: ('a, 'b) => 'c, fa: t('a), fb: t('b)): t('c) => {
  apply(map(f, fa), fb);
};
```

We call this `map2` because it looks a lot like `map` - it applies a pure
function to some independent values that are each inside their own
functor context.

```ocaml
let map: ('a => 'b, t('a)) => t('b);

let map2: (('a, 'b) => 'c, t('a), t('b)) => t('c);
```

It's important to note that the result of both of these is still inside our
functor context `t` - we don't have a way to get it out of that context. You
can also create `map3`, `map4`, etc. in the same way. These functions are
sometimes called `liftA2`, `liftA3`, etc., as they act to "lift" a pure
function `('a, 'b) => 'c` into the applicative context `(t('a), t('b)) =>
t('c)` (using the power of the `APPLY` typeclass).

For one final side note, we can also now implement our `tuple2` function in terms of
`map2`, and the same applies for `map3`/`tuple3`, etc.

```ocaml
let tuple2 = (fa: t('a), fb: t('b)): t(('a, 'b)) => map2((a, b) => (a, b), fa, fb);
```

These `mapN` functions are useful for running a bunch of independent
applicative effects and combining the results however we please.

# The map/ap/ap pattern

Let's get super fancy, and cast off our fear of weird operators, and do the
above using some Haskell-style infix operators. Once you grok the pattern
(and memorize the operators), the infix-based approach actually becomes
quite beautiful. The usual caveat with operators applies - they add a level
of opacity and abstraction that can be quite hostile to newcomers, so it's
best to introduce them with some hand-holding, and not just dump them on
people without the prerequisite setup.

Let's first define an operator for map. We're going to use `<$>` because it's
the conventional operator for `map` in many other FP languages, and it's
useful to start to recognize it for what it is.

```ocaml
let map: ('a => 'b, t('a)) => t('b) = ...;

let (<$>) = map;
```

Remember that the function is on the left, and the effectful value on the right, 
so you'd use this like so:

```ocaml
let f = a => a * 2;
let fa = Some(42);

// The following are all the same:

let _ = f <$> fa;
let _ = map(f, fa);
let _ = fa |> map(f);
```

Now let's define the operator `<*>` as `apply`, again the "function" part is
on the left (even though the function for `apply` is wrapped), and the
effectful value is on the right. Again, we're using `<*>` for `apply`,
because that's what many other languages do, and it helps to get used to it.

```ocaml
let apply: (t('a => 'b), t('a)) => t('b) = ...;

let (<*>) = apply;
```

Now let's take a look back at our `map3` function:

```ocaml
let map3 = (f: ('a, 'b, 'c) => 'd, fa: t('a), fb: t('b), fc: t('c)): t('d) => {
  apply(apply(map(f, fa), fb), fc);
};
```

Infix operators let you move the name of a function between the two
arguments, so we could write this like below - just start from the innermost
function application (`map`) and move the name of the function between the
args, and work your way out:

```ocaml
apply(apply(map(f, fa), fb), fc);

apply(apply(f `map` fa), fb, fc);

apply(f `map` fa `apply` fb, fc);

f `map` fa `apply` fb `apply` fc;
```

Now just replace `map` with `<$>` and `apply` with `<*>`:

```ocaml
let map3 = (f, fa, fb, fc) => f <$> fa <*> fb <*> fc;
```

If you read it left-to-right, we start with our function `f`, and we map it
over our first effectful value `fa: t('a)`, so now we have a wrapped
function, which we apply to our next effectful value `fb: t('b)`, and so on.
The two things below are the same thing - one just uses infix operators, and
the other uses normal named functions:

```ocaml
let _ = f <$> fa <*> fb <*> fc;

let _ = apply(apply(map(f, fa), fb), fc);
```

The operators work for any number of arguments, so we could implement `map4`,
`map5`, etc. like this too:

```ocaml
let map4 = (f, fa, fb, fc, fd) => f <$> fa <*> fb <*> fc <*> fd;

let map5 = (f, fa, fb, fc, fd, fe) => f <$> fa <*> fb <*> fc <*> fd <*> fe;

// etc.
```

This approach of applying a pure function to a series of "effectful" values
is very common in FP languages like Haskell and PureScript, so you'll likely
run into this at some point. I like to call this this **map/ap/ap** pattern,
because you just start with a pure function, you `map` it over the first
value, then you just `ap` it over all the remaining values.

You may run into the alternate form of this pattern, like this:

```ocaml
let map3 = (f, fa, fb, fc) => pure(f) <*> fa <*> fb <*> fc;
```

We've seen before how we can lift a pure function into our applicative
context using `pure` and then just `apply` it, so this is just another way of
achieving the same thing as **map/ap/ap**. The difference is that we
immediately lift our function into the applicative context, so we can't use
`map` (`<$>`) to apply it the first time - we have to go straight to `apply`
(`<*>`) for the first application, and for all the rest.

We'll see some more concrete uses of these operators below in "applicative
validation".

# Applicative "effects"

Let's take a break from code to talk about the concept of "effects" and
"effectful" values. These concepts are a bit overloaded in functional
programming, so let's take a look at a few examples. Let's assume our
functions can't throw exceptions, and we're in a language that doesn't have
the concept of a `null` value.

The function `'a => 'b` is an example of what appears to be a
pure/non-effectful function. Based on the types, there's no indication that
the function can fail to produce a value or fail for any other reason.
There's also no indication that the computation will be asynchronous - it
appears to just map an argument of type `'a` to a result value of type `'b`.

Let's now consider the function `'a => option('b)`. In this case, we are now
made aware (via the type system) that this function might either produce a
value of type `'b` (`Some(b)`), or might fail to produce a value (`None`). In
terms of "effects" we can say that this function has the "effect" of being
unable to produce a value in some, or possibly all cases. This isn't a "side
effect" like writing to STDOUT or reading the system time, but a behavior of
the function where we're no longer just mapping inputs to nice and clean
outputs. Just like `option`, it turns out we can actually model real-world
side effects via the type system, but we'll not get into that now.

How about the function `'a => Result.t('b, 'e)`? Here we can observe the
"effect" of possible failure - the function can either succeed and produce a
value of type `Ok(b)`, or fail and produce an error of type `Error(e)`. This
effect of possible failure is represented by the type `Result.t('a, 'e)`.

Let's quickly mention the function `'a => list('b)`. We've ignored
lists/arrays/trees/etc. to this point, but this type of function can also be
seen as an effect - the effect of indeterminate results. Here we could get no
results, a single result, or any number of results, and we can think about
this effect similarly to how we think about other effects.

Now consider the [RationalJS/future](https://gitub.com/RationalJS/future)
`Future` type. The way this type is defined and implemented, it represents an
asynchronous computation that cannot fail. Being asynchronous, it's possible
that it will just never complete, but it has no way of representing a
computation that completed but failed. We can call this an "async effect."
With this library, if you need to represent an asynchronous computation that
**can** fail, you're advised to use the type `Future.t(Result.t('a, 'e))` -
here we're combining the effect of possible failure with a separate async
effect. This is a good demonstration of how effects can be "stacked" via the
type system, and it's also a good example to show that it's possible to "run"
or "remove" a single effect, while leaving other effects intact for separate
or later processing. For example, if you were to allow the `Future` to
complete, you're given a value of type `Result.t('a, 'e)` which still has the
effect of possible failure. This failure effect can be "removed" by
attempting to `map` or `flatMap` the value, and handling the case of `Error`,
either by converting the error to a successful value, or handling it in some
other way.

The `Js.Promise.t('a)` type also has two effects: an async effect and the
effect of possibile failure. With this type, the possibility of failure is
not directly observable by the type alone. This is okay - it just means that
the type has an implicit/hidden/non-polymorphic way of representing the
failure condition, but there's nothing inherently wrong with that, you just
have to know that with a `Js.Promise`, there is a possibility of failure. You
might run into code that uses `Js.Promise.t(Result.t('a, 'e))`, but this is
problematic in that we now have two ways to represent failure - the "hidden"
`Js.Promise` error type, and the `'e` type from the `Result`. I hope to talk
about `Js.Promise` more in another blog post, but I'll leave it at that for
now.

As a final example, if we look at the JSON decoder type like the
`Decoder.t('a)` we defined above, we have a value that represents the effect
of parsing a JSON value into some type, and the effect of possible of failure
(which again is not represented by a polymorphic error type, but by a fixed
error type that we've defined with the decoder). This decoding effect is a
little different than the others in that it's more of a deferred computation,
but the same idea applies - the effectful value itself doesn't do anything
until we "run it" (by giving it a JSON value), and letting it produce a
`Result`, which is how the effect of possible failure manifests itself, and
can be handled or "run" separately from the decoding effect.

Before we move on, let's look back at the "pure" function `'a => 'b`. In many
languages, including ReasonML, this type of function can actually have all
sorts of effects which do not manifest themselves as "effectful values," but
just as plain old side effects. We can do I/O, read the system time, and even
launch the nukes, and nobody would be the wiser. It turns out you can
actually encapsulate these kinds of "side effects" in the type system using a
wide variety of different techniques, but we won't get into that now. The
topic of effect management is a very actively evolving and quite fascinating
topic in FP right now.

In summary, when we talk about applicatives (and monads), it's common to talk
about "effectful" values and "running" said effects. The concept of "running"
an effect sometimes can result in the "removal" of the effect, which
indicates that the effect has been handled, processed, or executed, but this
can mean different things for different types of effects. One key aspect of
functional programming is the separation of describing or encoding work to do
and the actual execution or interpretation of that work. If you're coming
from a more imperative language or style, where effects kind of just happen
when they happen, and are manifested by `null` values, exceptions, or
spaghetti code that just does whatever it wants whenever it wants, the
functional approach will take a little getting used to, but it unlocks a
great deal of control and power.

# Apply extensions

Now that we've seen some of the things that you can do with `APPLICATIVE`,
let's try to capture those ideas so we can reuse them for **every** applicative.
In the [functor article](/posts/2019-11-01-a-laymans-guide-to-functors-in-reasonml),
we saw that we could use a module functor to add "extensions" or "freebies"
for any instance of `FUNCTOR`, and now we'll do the same thing for `APPLY`.
Note that all of these extensions just need `APPLY` and not `APPLICATIVE`, because
these things only need `map` and `apply`, and not `pure`.

```ocaml
module ApplyExtensions = (A: APPLY) => {
  let applyFirst = (fa: A.t('a), fb: A.t('b)): A.t('a) => {
    let f = (a, _) => a; // const
    A.apply(A.map(f, fa), fb);
  };

  let applySecond = (fa: A.t('a), fb: A.t('b)): A.t('b) => {
    let f = (_, b) => b; // const(id)
    A.apply(A.map(f, fa), fb);
  };

  let map2 = (f: ('a, 'b) => 'c, fa: A.t('a), fb: A.t('b)): A.t('c) => {
    A.apply(A.map(f, fa), fb);
  };

  let map3 = (f: ('a, 'b, 'c) => 'd, fa: A.t('a), fb: A.t('b), fc: A.t('c)): A.t('d) => {
    A.apply(A.apply(A.map(f, fa), fb), fc);
  };

  // TODO: map4, map5, etc.

  let mapTuple2 = (f: ('a, 'b) => 'c, (fa: A.t('a), fb: A.t('b))): A.t('c) => {
    map2(f, fa, fb);
  };

  let mapTuple3 = (f: ('a, 'b, 'c) => 'd, (fa: A.t('a), fb: A.t('b), fc: A.t('c))): A.t('d) => {
    map3(f, fa, fb, fc);
  };

  // TODO: mapTuple4, mapTuple5, etc.

  let tuple2 = (fa: A.t('a), fb: A.t('b)) => map2((a, b) => (a, b), fa, fb);

  let tuple3 = (fa: A.t('a), fb: A.t('b), A.t('c)) => map3((a, b, c) => (a, b, c), fa, fb, fc);

  // TODO: tuple4, tuple5, etc. - as many as you want
};

module ApplyInfix = (A: APPLY) => {
  module AE = ApplyExtensions(A);

  let (<*>) = A.apply;

  let (<*) = AE.applyFirst;

  let (*>) = AE.applySecond;
};
```

There's a lot going on here, so let's break it down. We're creating a module
functor `ApplyExtensions` that takes an instance of `APPLY` and uses that
instance to define a bunch of extension functions. Because we have an `APPLY`
we are constrained to only having access to a type `A.t('a)`, the `A.map`
function, and the `A.apply` function, but we can do a lot with just these. Note
that you can see this in action in
[Relude_Extensions_Apply](https://github.com/reazen/relude/blob/master/src/extensions/Relude_Extensions_Apply.re).

We're first defining functions called `applyFirst` and `applySecond`. These
are interesting functions that take two effectful values, and **runs them
both**, but then only produces the result of the first or the second,
respectively. The key here is that we're actually running both effects, and
not just discarding the undesired side immediately - both effects must
succeed in order for us to get the result. If either or both effects fail, we
get an unsuccessful result, which can mean different things depending on
which `APPLY` instance we're using. E.g for `option`, the "unsuccessful
value" is `None`, and for `Result` the unsucessful value is `Error(e)`, etc.
These two functions are more commonly seen in their operator forms: `<*` and
`*>` they kind of look like half of the `apply` operator `<*>`, and point at
the argument that we want to keep. See
[ReludeParse](https://github.com/reazen/relude-parse) for a real-world
non-trivial use case for these operators. You can also think of these
operators as similar to `tuple2` where we first run and tuple the results,
then just take one side or the other.

Next we're defining a bunch of `mapN` functions - here we only go up to
`map2` and `map3`, but in your own library, you could go as high as you
wanted. Note that if you go above 5 or so arguments, you might be better off
just using the more flexible **map/ap/ap** pattern with `<$>` and `<*>`
operators. If you were curious, the reason we need all these numbered map and
other functions, rather than operating on lists or arrays of effectful
values, it's because lists and arrays are homogenous - they can only carry
values of the same type, but we want to operate on values of different types.
In a later blog post, we'll see a new abstraction called `TRAVERSABLE` which
can help us deal with lists/arrays/options/etc. that containing effectful
values.

We then define an alternative version of `mapN` called `mapTupleN` - this
function simply takes a tuple of the effectful values, and then just
destructures the tuple and runs them through `mapN`. Using a tuple can
sometimes be a more convenient way of collecting and chaining a series of
operations, e.g.

```ocaml
(Some(42), Some("hi"), Some(bool))
|> Option.mapTuple3((i, s, b) => ...do something here...);
```

Finally we create a separate module functor for defining infix operators.
Having a separate module for infix operators can be handy for when you want
to do a local open and just get access to the operators for a few small
operations. Note that using the OCaml/ReasonML `include` mechanism, you can
choose to include the infix operators into any organizational module you
want.

The pattern we use in Relude to incorporate these extensions is something
like this:

```ocaml
module Option = {
  type t('a) = option('a);

  let map = ...;

  let apply = ...;

  let pure = ...;

  module Functor: FUNCTOR = {
    type nonrec t('a) = t('a); // alias
    let map = map; // alias
  };
  include FunctorExtensions(Functor);

  module Apply: APPLY = { ... };
  include ApplyExtensions(Apply);

  module Applicative: APPLICATIVE = { ... };
  include ApplicativeExtensions(Applicative);

  // ... other typeclasses and extension module functors

  module Infix = {
    include FunctorInfix(Functor);
    include ApplyInfix(Functor);
    // ... other infix module functors
  };
};
```

This way, all the top-level functions are exposed at the top level of
the module for convenience, then we have the typeclasses also at the top level
with their corresponding functions and types defined as just aliases, and we
immediately construct and include each of the typeclass extension modules.
This include puts the extension functions like `map3`, `tuple3`, etc. right
at the top-level of the module, so you can do things like `Option.map3(...)`.
Finally, we create a separate `Infix` module, where we include all of the 
`Infix` extension modules. This puts all the operators in one common scope,
so you can do things like this:

```ocaml
let x = Option.Infix.(
  f <$> Some(42) <*> Some("hi") <*> Some(true)
);
```

For types with more than one type parameter (like `Result.t('a, 'e)`), we
unfortunately have to implement all of our typeclass instances and extensions
inside a module functor (like `Result.WithError`), so we lose a little of the
convenience of the extensions. In order to get access to `map3`, etc. for `Result`,
you have to do something like this:

```ocaml
module ResultE = Result.WithError({ type t = myErrorType });

ResultE.map3(...);
```

Unfortunately, you can't inline module functor stuff with function invocations,
so you can't do this, which is a big bummer:

```ocaml
// Can't do this :(
let x = Result.WithError({ type t = string }).map3(...);
```

Also, as we saw in the functor article, you can't pass modules that deal with
higher-kinded types via first-class modules, so we're kind of stuck with the
extra boilerplate of instantiating our module in one line, and using it where
we need it.

That was a lot of discussion, but in case you missed it, we just gave
ourselves an implementation of `mapN`, `tupleN`, `mapTupleN`, `<*>`, `<*`,
`*>` for **any module that has an `APPLY` instance**! I think that's pretty
awesome! Your `option`, `Result`, `Js.Promise`, `Future`, `IO`, and every
other `APPLY` gets a handy, consistent and powerful set of functions for
free, just because we took the time to implement `map` and `apply` and setup
some typeclass instances. Plus, we now have a layer of centralized
abstraction and extension where we can add more of these types of functions,
and we just get them all for free anywhere we're using `ApplyExtensions`! If
you think this is cool, you should take a look at languages like Haskell or
PureScript and see how they do all this with much less ceremony, but it's
still pretty cool that we can achieve something very similar in ReasonML!

# Applicative validation

We now have a bunch of useful tools for dealing with `APPLY` and
`APPLICATIVE` instances and their "effectful values," so let's see what we
can do with them.

First of all, if you've used `Promise.all` in JavaScript (or the
`Js.Promise.allN` equivalents in ReasonML), you've already used an
applicative-style API. We have our own versions of these in our extensions
like `mapN`, `tupleN`, `mapTupleN`, etc. These are all just helper functions
that let us process a bunch of effectful values in parallel and combine the
results in different ways, which is quite handy all by itself.

If you have a situation where you have a bunch effectful values to deal with
(say `N+1` values), and your `mapN` functions only "go up to `N`," you can
deal with an arbitrary number of arguments using the **map/ap/ap** pattern
using `<$>` and `<*>`, which we'll see below.

Let's now look at one more very useful use case for applicatives: applicative
validation, parsing, and decoding. But first, let's introduce a few additional
concepts to help us out:

# SEMIGROUP typeclass

Now that we've seen `FUNCTOR`, `APPLY`, and `APPLICATIVE`, we know that
typeclasses are sort of just sets of functions that must conform to some
laws, so I think we can start to introduce more of these things without fear.
Just look at the types of the functions and the laws - you don't necessarily
need to understand why it exists or what it's for right away, just see it for
what it is: functions and laws.

The `SEMIGROUP` typeclass is simply about the ability to combine values of the
same type:

```ocaml
module type SEMIGROUP = {
  type t;
  let append: (t, t) => t;
};
```

The `append` function often has an operator version too - in Haskell, it's
`<>`, but we'll ignore that for now (other than for noting the law). The name
`append` is commonly used, but you could also think of this operation as
`combine`.

The laws for this typeclass is that the `append` function must be associative,
i.e.

```ocaml
// Associativity (<> is the operator for append)

(a <> b) <> c == a <> (b <> c)
```

If you remember from algebra class, it's the same law of associativity that
we learned for addition and multiplication. Basically, it requires that if
you have 3 values, you can combine the first two first, then combine the
third and that should be the same as combining the second two, then combining
that with the first. Note that this is not commutativity - the items have to
be combined in the order above, we're not saying that you can switch the
order of the arguments, we're just saying exactly what's described above.
There is another typeclass that requires commutativity, but `SEMIGROUP`
doesn't - only associativity.

```ocaml
// Commutativity - this is NOT a law required by SEMIGROUP
a <> b == b <> a
```

Let's see a few quick examples for `int` addition, `int` multiplication,
`bool` AND and OR, `string`, and `list('a)`.

Integers actually have multiple semigroups. One semigroup is integer
addition, where the `append` operation is just `+`. Another different
semigroup is multiplication where the `append` function is `*`. You might
hear these notions described as something like "integers form a semigroup
under addition."

```ocaml
module Integer = {
  module Addition = {
    module Semigroup: SEMIGROUP = {
      type t = int;
      let append = (a, b) => a + b;
    };
  };

  module Multiplication = {
    module Semigroup: SEMIGROUP = {
      type t = int;
      let append = (a, b) => a * b;
    };
  };
};
```

We define these as distinct instances, because they are distinct semigroups.
Note that integer subtraction and division are **not SEMIGROUPS** because
those operations are not associative (they don't conform to the semigroup
law, which requires that the append operation be associative). These
operations might be usuable for other typeclasses, but not `SEMIGROUP`.

```ocaml
(1-2)-3 != 1-(2-3)
(1/2)/3 != 1/(2/3)
```

Boolean has a few semigroups too: the `AND` and `OR` operators. These would
be implemented similarly to how I split up `Addition` and `Multiplication`
for `Integer`, and for booleans, you'll see words like `Conjunctive` for
`AND` and `Disjunctive` for `OR`.

Strings have a SEMIGROUP too, just string concatenation:

```ocaml
let append = (str1, str2) => str1 ++ str2;
```

Lists and arrays also form a semigroup, but since these are type constructors
of the form `t('a)`, we need a variation of `SEMIGROUP` that works with types
of that form. We'll call it `SEMIGROUP_ANY` - that's what it's called in
[bs-abstract](https://github.com/Risto-Stevcev/bs-abstract).

```ocaml
module type SEMIGROUP_ANY = {
  type t('a);
  let append: (t('a), t('a)) => t('a);
};
```

The semigroup for `list` and `array` is just concatenation, just like
`string`. Here's a quick implementation for `list('a)`

```ocaml
module List = {
  type t('a) = list('a);

  let concat = (list1, list2) =>
    Belt.List.concat(list1, list2); // I hope I got the order right for Belt.List...

  module SemigroupAny: SEMIGROUP_ANY = {
    type nonrec t('a) = t('a);
    let append = concat;
  };
};
```

I hope that's clear enough for now, and let's move on.

# NonEmptyList data type

Let's now introduce a useful type for error handling - a `NonEmptyList`.
Sometimes when you're dealing with collections of values, it can be really
helpful to guarantee that there's at least one value. Imagine a validation
function in JavaScript, where maybe it gives you back an array of errors -
what happens if it returns you an empty array? That seems odd and
undesirable, so let's not allow ambiguous things like that by using better
types. Just for expedience, I'm going to go ahead and implement this now,
including a `SemigroupAny` instance - below is how we typically implement
new data types in [Relude](https://github.com/reazen/relude):

```ocaml
module NonEmptyList = {
  type t('a) = | Nel('a, list('a));

  let map = ...;

  let apply = ...;

  let pure = a => Nel(a, []);

  let concat = (Nel(h1, t1), Nel(h2, t2)) => {
    Nel(h1, Belt.List.concat(t1, [h2, ...t2]));
  };

  module SemigroupAny: SEMIGROUP_ANY = {
    type nonrec t('a) = t('a);
    let append = concat;
  };
  include SemigroupAnyExtensions(SemigroupAny);

  module Functor: FUNCTOR with type t('a) = t('a) = ...
  include FunctorExtensions(Functor);

  module Apply: APPLY with type t('a) = t('a) = ...
  include ApplyExtensions(Apply);

  module Applicative: APPLY with type t('a) = t('a) = ...
  include ApplicativeExtensions(Applicative);
};
```

The `NonEmptyList` is basically just a data structure with a constructor
`Nel` that has one guaranteed value `'a`, and a "tail" `list('a)`, which can
be empty. We can implement `map`, `apply`, `pure`, `concat` and many other
useful things for this type, and all the corresponding typeclass instances.

There's nothing magic about any of this - it's simply a type that is exactly
what it says it is: a non-empty list. We're going to use this below for
"collecting errors" in our applicative validation, so that we can be sure
that if an applicative operation fails, we'll get at least one error value
out.

# Validation data type

If you recall above when we implemented `apply` for `Result.t('a, 'e)`, we
ran into a conundrum when faced with the "both sides failed" case:

```ocaml
let apply = (ff: Result.t('a => 'b, 'e), fa: Result.t('a, 'e)) => switch(ff, fa) {
  | (Ok(aToB), Ok(a)) => Ok(aToB(a))
  | (OK(aToB), Error(e2)) => Error(e2)
  | (Error(e1), Ok(a)) => Error(e1)
  | (Error(e1), Error(e2)) => Error(e1) // !!!
}
```

In this `Error/Error` case, we have no way to **combine** the errors, and no
real way to decide which one to return, so we'll just arbitrarily return the
left side's error.

Let's introduce a new data type called `Validation.t('a, 'e)` to deal with
this case specifically. We're going to add a little extra spice to our
`apply` implementation to deal with these two errors. The type itself is
going to look almost exactly the same as `Result.t('a, 'e)`, but it will
have different semantics for the applicative behavior.

```ocaml
module Validation = {
  type t('a, 'e) = | VOk('a) | VError('e);

  let map = (aToB, fa) => switch(fa) {
    | VOk(a) => VOk(aToB(a))
    | VError(e) => VError(e)
  };

  let applyWithCombineErrors = (combineErrors: ('e, 'e) => 'e, ff: t('a => 'b, 'e), fa: t('a, 'e)): t('b, 'e) => {
    switch(ff, fa) {
      | (VOk(aToB), VOk(a)) => VOk(aToB(a))
      | (VError(e1), VOk(a)) => VError(e1)
      | (VOk(aToB), VError(e2)) => VError(e2)
      | (VError(e1), VError(e2)) => VError(combineErrors(e1, e2)) // :tada:
    };
  };

  let pure = a => VOk(a);
};
```

To implement our typeclass instances like `Functor`, etc. we'll use the
`WithError` module functor trick to "lock down" our error type. However, when
we get to `APPLY`, where is this `combineErrors` function going to come from?
We don't know what the error type is (well, we'll know once we lock it down
using `WithError`), and moreover, we don't know how to combine these errors.
Oftentimes, when you need to do something, and you don't know how to do it,
you can just make someone pass it to you. In this case, we need a mechanism
for combining values of the same type, and for this, we have the `SEMIGROUP`
abstraction! Let's see how this works:

```ocaml
module Validation = {
  type t('a, 'e) = | VOk('a) | VError('e);

  let map = (aToB, fa) => switch(fa) {
    | VOk(a) => VOk(aToB(a))
    | VError(e) => VError(e)
  };

  let applyWithCombineErrors = (combineErrors: ('e, 'e) => 'e, ff: t('a => 'b, 'e), fa: t('a, 'e)): t('b, 'e) => {
    switch(ff, fa) {
      | (VOk(aToB), VOk(a)) => VOk(aToB(a))
      | (VError(e1), VOk(a)) => VError(e1)
      | (VOk(aToB), VError(e2)) => VError(e2)
      | (VError(e1), VError(e2)) => VError(combineErrors(e1, e2)) // :tada:
    };
  };

  let pure = a => VOk(a);

  module WithErrors = (Errors: SEMIGROUP_ANY, Error: TYPE) => {
    module Functor: FUNCTOR with type t('a) = t('a, Errors.t(Error.t)) = {
      type nonrec t('a) = t('a, Errors.t(Error.t));
      let map = map;
    };
    include FunctorExtensions(Functor);

    module Apply: APPLY with type t('a) = t('a, Errors.t(Error.t)) = {
      include Functor;
      let apply = (ff, fa) => applyWithCombineErrors(Errors.append, ff, fa);
    };
    include ApplyExtensions(Apply);

    module Applicative: APPLICATIVE with type t('a) = t('a, Errors.t(Error.t)) = {
      include Apply;
      let pure = pure;
    };
    include ApplicativeExtensions(Applicative);

    module Infix = {
      include FunctorInfix(Functor);
      include ApplyInfix(Apply);
    };
  };
};
```

There are multiple ways of doing this, but I'm just going to require a
`SEMIGROUP_ANY` instance to collect errors, and a `TYPE` instance to lock
down my error type. You could also probably use a plain `SEMIGROUP`, but that
requires some different techniques for dealing with the error type.

Now in order to use the applicative stuff with this type, I just need to give
it a `SEMIGROUP_ANY` and an error `TYPE`, like so:

```ocaml
module ValidationNel = Validation.WithErrors(NonEmptyList.SemigroupAny, { type t = string });

ValidationNel.tuple3(VOk(42), VOk("hi"), VOk(true)) // VOk((42, "hi", true))

ValidationNel.tuple3(VOk(42), VError("oops"), VError("darn")) // VError(Nel("oops", ["darn"]))
```

With this type and the `SEMIGROUP_ANY` semantics, we have the ability to
collect the error for each individual failure in the parallel operation,
rather than just exploding at the first error we encounter! To understand how
this works, just think about how `apply` is implemented for `Validation`,
especially in the case of both sides failing. We can use a `NonEmptyList` to
collect our errors, because there's never a scenario where this can fail and
we don't have an error value to deal with. We could also use a plain old
`list('a)`, but then you lose the guarantee at the callsite of there being at
least one error - the caller has to deal with the empty list case, when we
know that will never actually happen. This is all about [making impossible states
impossible](https://www.youtube.com/watch?v=IcgmSRJHu_8).

# Applicative validation with error collection

We now have a type `Validation.t('a, 'e)` that looks a lot like a `Result.t('a, 'e)`,
but has slightly different semantics in its `APPLICATIVE` instance. The `Result.t('a, 'e)`
`apply` function "fails fast" in that it just propagates the first error it encounters
all the way to the final result, whereas the `Validation` has "error collecting" semantics
where it will collect and combine each error it encounters using a `SEMIGROUP` or in our
case a `SEMIGROUP_ANY` instance, which is just a semigroup for types of the form `t('a)`.

Note that when I say "Result fails fast" in its applicative, I don't mean it
instantly explodes or early-returns out of the function. I just mean that if
some individual operation fails, the `apply` function will just propagate
that error along, and ignore any further errors, until it reaches the final
result - just think about how `apply` is implemented for `Result`.

We can use applicatives for a wide variety of validation, decoding, parsing,
and data normalization tasks. Some real world use cases for applicative
validation might include CSV parsing, JSON decoding, web form validation,
string parsing, database record parsing, parallel async data fetching, and
countless more. Let's see a few examples, but for all of these remember that
you can use any type that has an applicative instance to do the same thing
but with different error handling semantics, and in fact, you can abstract
the `APPLICATIVE` instance away using a module functor to make a validation
module that can just work with any applicative right off the bat. When I say
"desired semantics" I just mean that you can choose the type of error
handling that fits your needs - if you just want to know whether the
validation succeeded or failed, you could use `option('a)`, if you wanted to
know whether it failed, and what the first error was, you might use
`Result.t('a, 'e)` (which doesn't know how to combine the errors in `apply`),
or if you wanted to capture all the errors, you might use a `Validation.t('a,
'e)`. If your validation requires async computations, you might need to use
something like `Promise.t('a)` or `Future.t(Result.t('a, 'e))`, or even
`Future.t(Validation.t('a, 'e))`.

Anyway, let's see a quick example. Let's say we have a model like the following:

```ocaml
module User = {
  type t = {
    name: string,
    email: string,
    age: int,
    isAdmin: bool
  };

  let make = (
    name: string,
    email: string,
    age: int,
    isAdmin: bool
  ) => {name, email, age, isAdmin};
```

We have a type `User.t` which has information about users, and a pure `make` function which
simply acts as a non-validating constructor for the type `User.t`. Let's say we're getting a
JSON value representing a user, and we want to decode it into a value of type `User.t`, so that
we can be confident that we know what data we're working with.

Let's build a tiny JSON decoder library based on `Validation.t('a, 'e)`, and then see how we
can implement a basic JSON decoder using Validation, and the JSON decoder type we saw above.
Here is the full example, and I'll explain more below.

```ocaml
module type TYPE = {type t;};

module type SEMIGROUP_ANY = {
  type t('a);
  let append: (t('a), t('a)) => t('a);
};

module type FUNCTOR = {
  type t('a);
  let map: ('a => 'b, t('a)) => t('b);
};

module FunctorExtensions = (F: FUNCTOR) => {
  let voidLeft = (b: 'b, fa: F.t('a)) => F.map(_ => b, fa);

  let voidRight = (fa: F.t('a), b: 'b) => F.map(_ => b, fa);
  // etc.
};

module FunctorInfix = (F: FUNCTOR) => {
  module FE = FunctorExtensions(F);

  let (<$>) = F.map;

  let (<$) = FE.voidLeft;

  let ($>) = FE.voidRight;
  // etc.
};

module type APPLY = {
  include FUNCTOR;
  let apply: (t('a => 'b), t('a)) => t('b);
};

module ApplyExtensions = (A: APPLY) => {
  let map2 = (f, fa, fb) => A.apply(A.map(f, fa), fb);

  let map3 = (f, fa, fb, fc) => A.apply(A.apply(A.map(f, fa), fb), fc);

  let map4 = (f, fa, fb, fc, fd) =>
    A.apply(A.apply(A.apply(A.map(f, fa), fb), fc), fd);

  let map5 = (f, fa, fb, fc, fd, fe) =>
    A.apply(A.apply(A.apply(A.apply(A.map(f, fa), fb), fc), fd), fe);
  // etc.
};

module ApplyInfix = (A: APPLY) => {
  let (<*>) = A.apply;
  // etc.
};

module type APPLICATIVE = {
  include APPLY;
  let pure: 'a => t('a);
};

module NonEmptyList = {
  type t('a) =
    | Nel('a, list('a));

  let pure = a => Nel(a, []);

  let concat = (Nel(h1, t1), Nel(h2, t2)) => {
    Nel(h1, List.concat([t1, [h2], t2]));
  };

  let toList = (Nel(h, t)) => [h, ...t];

  let toArray = nonEmptyList => nonEmptyList |> toList |> Belt.List.toArray;

  module SemigroupAny: SEMIGROUP_ANY with type t('a) = t('a) = {
    type nonrec t('a) = t('a);
    let append = concat;
  };
};

module Validation = {
  type t('a, 'e) =
    | VOk('a)
    | VError('e);

  let ok = a => VOk(a);

  let error = e => VError(e);

  let errorNel = e => VError(NonEmptyList.pure(e));

  let map = (f, fa) =>
    switch (fa) {
    | VOk(a) => VOk(f(a))
    | VError(e) => VError(e)
    };

  let applyWithAppendErrors =
      (appendErrors: ('e, 'e) => 'e, ff: t('a => 'b, 'e), fa: t('a, 'e)) =>
    switch (ff, fa) {
    | (VOk(f), VOk(a)) => VOk(f(a))
    | (VError(e1), VOk(_)) => VError(e1)
    | (VOk(_), VError(e2)) => VError(e2)
    | (VError(e1), VError(e2)) => VError(appendErrors(e1, e2))
    };

  let pure = ok;

  module WithErrors = (S: SEMIGROUP_ANY, E: TYPE) => {
    type nonrec t('a) = t('a, S.t(E.t));

    module Functor: FUNCTOR with type t('a) = t('a) = {
      type nonrec t('a) = t('a);
      let map = map;
    };
    let map = map;
    include FunctorExtensions(Functor);

    module Apply: APPLY with type t('a) = t('a) = {
      include Functor;
      let apply = (ff, fa) => applyWithAppendErrors(S.append, ff, fa);
    };
    let apply = Apply.apply;
    include ApplyExtensions(Apply);

    module Applicative: APPLICATIVE with type t('a) = t('a) = {
      include Apply;
      let pure = pure;
    };
    let pure = pure;

    module Infix = {
      include FunctorInfix(Functor);
      include ApplyInfix(Apply);
    };
  };
};

module Decode = {
  module Error = {
    type t =
      | ExpectedString(Js.Json.t)
      | ExpectedBool(Js.Json.t)
      | ExpectedInt(Js.Json.t)
      | ExpectedObject(Js.Json.t)
      | ExpectedObjectWithKey(Js.Json.t, string);

    let show = e =>
      switch (e) {
      | ExpectedString(json) =>
        "Expected string, got: " ++ Js.Json.stringify(json)
      | ExpectedBool(json) =>
        "Expected bool, got: " ++ Js.Json.stringify(json)
      | ExpectedInt(json) => "Expected int, got: " ++ Js.Json.stringify(json)
      | ExpectedObject(json) =>
        "Expected object, got: " ++ Js.Json.stringify(json)
      | ExpectedObjectWithKey(json, key) =>
        "Expected object with key "
        ++ key
        ++ ", got: "
        ++ Js.Json.stringify(json)
      };
  };

  // For this example, we're going to use `Validation.t('a, NonEmptyList.t(Error.t))`
  // as our applicative "effect" type for the decoder. This will give us error-collecting
  // semantics in the decode operation.

  module ValidationE =
    Validation.WithErrors(NonEmptyList.SemigroupAny, Error);

  // Our decoder type is a function Js.Json.t => A.t('a)
  // Think of `A` being option, Result, Validation, Promise, etc. but here we're just
  // specialized to using ValidationE

  type t('a) =
    | Decode(Js.Json.t => ValidationE.t('a));

  // define map/Functor

  let map = (f, Decode(jsonToFA)) =>
    Decode(json => jsonToFA(json) |> ValidationE.map(f));

  module Functor: FUNCTOR with type t('a) = t('a) = {
    type nonrec t('a) = t('a);
    let map = map;
  };
  include FunctorExtensions(Functor);

  // define apply/Apply

  let apply = (Decode(jsonToFAToB), Decode(jsonToFA)) =>
    Decode(
      json => {
        let fAToB = jsonToFAToB(json);
        let fA = jsonToFA(json);
        ValidationE.apply(fAToB, fA);
      },
    );

  module Apply: APPLY with type t('a) = t('a) = {
    include Functor;
    let apply = apply;
  };
  include ApplyExtensions(Apply);

  // define pure/Applicative

  let pure = a => Decode(_ => ValidationE.pure(a));

  module Applicative: APPLICATIVE with type t('a) = t('a) = {
    include Apply;
    let pure = pure;
  };

  // Add an infix operator module

  module Infix = {
    include FunctorInfix(Functor);
    include ApplyInfix(Apply);
  };

  // Now define some specific types of decoders

  let string: t(string) =
    Decode(
      json =>
        switch (Js.Json.classify(json)) {
        | JSONString(str) => ValidationE.pure(str)
        | _ => Validation.errorNel(Error.ExpectedString(json))
        },
    );

  let bool: t(bool) =
    Decode(
      json =>
        switch (Js.Json.classify(json)) {
        | JSONTrue => VOk(true)
        | JSONFalse => VOk(false)
        | _ => Validation.errorNel(Error.ExpectedBool(json))
        },
    );

  let int: t(int) =
    Decode(
      json =>
        switch (Js.Json.classify(json)) {
        | JSONNumber(f) => VOk(Js.Math.floor(f))
        | _ => Validation.errorNel(Error.ExpectedInt(json))
        },
    );

  let field = (key: string, decode: t('a)): t('a) =>
    Decode(
      json =>
        switch (Js.Json.classify(json)) {
        | JSONObject(dict) =>
          switch (Js.Dict.get(dict, key)) {
          | Some(value) =>
            let Decode(decodeA) = decode;
            decodeA(value);
          | None =>
            Validation.errorNel(Error.ExpectedObjectWithKey(json, key))
          }
        | _ => Validation.errorNel(Error.ExpectedObject(json))
        },
    );

  let run = (json: Js.Json.t, Decode(f)) => f(json);
};

module User = {
  type t = {
    name: string,
    email: string,
    age: int,
    isAdmin: bool,
  };

  let make = (name, email, age, isAdmin) => {name, email, age, isAdmin};

  let decode =
    Decode.Infix.(
      make
      <$> Decode.field("name", Decode.string)
      <*> Decode.field("email", Decode.string)
      <*> Decode.field("age", Decode.int)
      <*> Decode.field("isAdmin", Decode.bool)
    );

  // Note: could also use map4 rather than <$>/<*>

};

let json1 =
  Js.Json.object_(
    Js.Dict.fromList([
      ("name", Js.Json.string("Andy")),
      ("email", Js.Json.string("test@example.com")),
      ("age", Js.Json.number(55.0)),
      ("isAdmin", Js.Json.boolean(true)),
    ]),
  );

let validatedUser1 = User.decode |> Decode.run(json1);

switch (validatedUser1) {
| VOk(user) => Js.log2("Success!", user)
| VError(errors) => Js.log2("Failure!", errors |> NonEmptyList.toArray)
};

// Success! [ 'Andy', 'test@example.com', 55, true ]

let json2 =
  Js.Json.object_(
    Js.Dict.fromList([
      ("name", Js.Json.string("Andy")),
      ("email", Js.Json.number(42.0)),
      ("age", Js.Json.number(55.0)),
      ("isAdmin", Js.Json.string("supposed to be a boolean")),
    ]),
  );

let validatedUser2 = User.decode |> Decode.run(json2);

switch (validatedUser2) {
| VOk(user) => Js.log2("Success!", user)
| VError(errors) =>
  Js.log2(
    "Failure!",
    errors |> NonEmptyList.toArray |> Array.map(Decode.Error.show),
  )
};

// Failure! [ 'Expected string, got: 42',
//            'Expected bool, got: "supposed to be a boolean"' ]
```

So that was a long code sample, but most of it was just setting up all the
core pieces that would normally come from a library like
[Relude](https://github.com/reazen/relude) - I'm just showing it here to have
a complete and somewhat realistic example for demonstrating all the moving
parts.

- At the top of the file, we have all the core typeclass definitions
  - `TYPE`
  - `SEMIGROUP_ANY`
  - `FUNCTOR`
  - etc.
- We also have a couple `Extensions` and `Infix` modules for a few of these,
  which we'll use below.
- Next we define out `NonEmptyList` type, which we use in the `Validation` to
  collect the errors in the `apply` function
- Then we have the `Validation` module, with the core type, functions, then
  the typeclass instances wrapped inthe `WithErrors` module functor, where 
  we require a `SEMIGROUP_ANY` instance to collect our errors, and a `TYPE`
  instance to lock down the error type.
- Lastly for the "library" pieces, we define a `Decoder` module, which is
  essentially a function `Js.Json.t => Validation.t('a, NonEmptyList.t(Error.t))`,
  and all the supporting functions and typeclass instances.

As for the actual demonstration, I have a `User` module which has:

- a basic record type with 4 fields
- a pure, non-validating `make` function which acts as a constructor
- a decoder, which uses the applicative **map/ap/ap** pattern to map the
  pure `make` function over a series of decoders for the individual fields.
  - Note that instead of `<$>`/`<*>` we could use the `Decode.map4` function
   which we got "for free" from our `Decode.Apply` and the `include ApplyExtensions(Apply)`

The `Decoder` itself has an applicative instance, which is backed by the
applicative for the `Validation` module, which has error collecting
semantics. If you want to try an exercise, see if you can make the decoder
a module functor like `module WithApplicative => (A: APPLICATIVE, ???) =>`
and replace all usages of `Validation`/`ValidationE` with just `A` references.
Note: you'll need some way to deal with the errors, like the places where
I have `Validation.errorNel(...)`.

Anyway, that was a long example, with possibly insufficient explanation, but
I hope that by providing a complete, working example, anyone who is
interested can dig in more.

If you've used Elm, you might have recognized this type of pattern from
the [elm-decode-pipeline](https://package.elm-lang.org/packages/NoRedInk/elm-decode-pipeline/latest/)
library. That library is more built-out, but it is fundamentally built around
the idea of applicatives and parallel decoding of values.

You can also see this same pattern in ReasonML in the
[bs-decode](https://github.com/mlms13/bs-decode) library. This library is also
more robust than our simple example here, but the applicatives are there at
the core!

# The applicative laws

As I mentioned in the functor article, the typeclass laws are quite
important, but given that this article is already quite long, I'm just going
to link to the laws on a couple different sites - I encourage you to check
these out on your own:

- [Haskell Wikibooks Applicative functor laws](https://en.wikibooks.org/wiki/Haskell/Applicative_functors)
- [Applicative functors (using Flow type annotation)](https://medium.com/@JosephJnk/an-introduction-to-applicative-functors-aea966799b1d)

# Conclusion

I hope you enjoyed my article on applicative functors - let me know if you
find any problems with the code samples, or if you have any questions or
suggestions.