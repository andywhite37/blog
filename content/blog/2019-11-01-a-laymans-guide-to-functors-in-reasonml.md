---
title: "A Layman's Guide to Functors in ReasonML"
date: 2019-11-01T17:50:46-06:00
draft: false
---

In [my previous post](/blog/2019-10-31-hello-world) I talked about my
background and how I started on my journey to learn typed functional
programming. I'll again preface these posts with a note that I don't have a
background in category theory, so these posts are intended to help introduce
things from a boots-on-the-ground perspective. Please feel free to correct me
on any points I've messed up.

# Functors

In this post, I would like to talk about functors - not [OCaml/ReasonML
module functors](https://v1.realworldocaml.org/v1/en/html/functors.html), but
functors as they relate to the `map` function. I would wager that most
software developers have probably used a `map` function for lists or arrays,
and probably a map-like function when dealing with asynchronous types like
`Future` or `Promise`, to convert a value of some type `A` to a value of some
other type `B`. If you're lucky, you've used `map` on a type like `Option`,
`Maybe`, `Either` or `Result`. If you're really blessed, maybe you've used it
in something like a JSON decoder function to transform a decoded number into
some other type.

# ReasonML - a quick aside

I'm going to use the language [ReasonML](https://reasonml.github.io) for
these posts, not because it's the most powerful language around (it's not),
but because it provides an interesting platform on which to build up some
examples. If you're not familiar with ReasonML, it's basically an alternative
syntax for the programming language [OCaml](https://ocaml.org). The ReasonML
programming language can be losslessly translated to OCaml and vice-versa,
because the two languages share the same [abstract syntax tree
(AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) - the data
structure that represents the parsed code.

One of the reasons ReasonML was created was to provide a syntax that would be
more familiar to developers coming from JavaScript/ECMAScript. ReasonML has a
few different compilers which are able to produce JavaScript as a "backend,"
meaning that it can transpile ReasonML (or OCaml) code to JavaScript, for use
in the browser, or in Node.js, for example. The compiler I most use at the
moment is called [BuckleScript](https://bucklescript.github.io/en/), which is
actually specifically an OCaml to JS compiler. To make it work with ReasonML,
you use a separate tool to translate the ReasonML code to OCaml code before
compiling it with the BuckleScript compiler to produce the JavaScript output,
which you then have to bundle with a tool like [Webpack
:scream:](https://webpack.js.org), [Parcel](https://parceljs.org),
[Rollup](https://rollupjs.org/guide/en/) or whatever the latest
*so hot right now* bundler happens to be in the JavaScript community.

The workflow is basically:

1. Write ReasonML code
2. Convert ReasonML code to OCaml
3. Compile OCaml code with BuckleScript to produce JavaScript code
4. Bundle JavaScript code to produce something you can use on the web or in Node.js

If this sounds a little crazy to you, you're right, it is a little crazy.
However steps 2 and 3 kind of happen at the same time by the same tool, and
the BuckleScript compiler is blazing fast, so if you're writing ReasonML, you
really don't see OCaml code at all, and the JavaScript that comes out the
other end is quite readable and usually pretty optimized. Overall, there is
some weight of tooling with this, but the installation of these tools have
been mostly ironed out, and are improving each day.

As a side note, you can also compile ReasonML code to native code using the
native OCaml compiler and tools like [esy](https://esy.sh) or
[dune](https://github.com/ocaml/dune). I don't have any experience with these
native tools, so I won't say anymore about that.

# Shameless plug

As a side note, I currently work on a set of libraries for
ReasonML/BuckleScript in the [Reazen GitHub org](https://github.com/reazen).
Our core "standard library replacement" library is called
[Relude](https://github.com/reazen/relude), and there are a growing number of
other libraries built on top of Relude. Many of the ideas from these blog
posts can be seen in action in Relude, and it's underlying library
[bs-abstract](https://github.com/Risto-Stevcev/bs-abstract), so check them
out if you are interested. Also, I'll re-emphasize that most of the ideas in
Relude are not our own - they come from the rich history and ecosystem of
typed functional programming.

# Back to functors

To jump right in, I'm going to introduce one way to represent a functor in
ReasonML and then go from there. In ReasonML, we have the concept of
[modules](https://ocaml.org/learn/tutorials/modules.html), which are a
fundamental and first-class structure in the language. You can think
of modules as being structural and organizational, like a namespace, but
modules can also be passed to functions as a first-class construct in the
language. You can construct modules from other modules using [module
functors](https://v1.realworldocaml.org/v1/en/html/functors.html), but as a
reminder, these are not the functors we're talking about in this blog. You
can also create [module
types](https://ocaml.org/learn/tutorials/modules.html), which sort of act as
a way to describe an "interface" for a module. If none of these words mean
anything to you, I'd recommend reading some more about OCaml or ReasonML
modules before continuing, otherwise this might quickly become confusing and
unfamiliar.

A functor has a couple key properties which we can encode in ReasonML using a
module type. One key aspect of a functor is that it deals with types that
have a single type parameter, e.g.

- `list('a)`
- `array('a)`
- `option('a)`,
- `Js.Promise.t('a)`
- `Belt.Result.t('a, fixedErrorType)` if the error type is fixed to a known type
- and many others, both generic and domain-specific

In general, we'll just call types of this form `t('a)`.

The next requirement of a functor is that it must support a `map` function
with the following type signature:

```ocaml
let map = ('a => 'b, t('a)) => t('b);
```

Basically, `map` is a function that accepts an `'a => 'b` function to convert
a value of type `'a` to a value of type `'b`, a value of type `t('a)`, and
returns to you a value of type `t('b)`.

Another way to look at `map` is that it takes a function of type `'a => 'b`
and returns to you a function of type `t('a) => t('b)`. ReasonML is a
[curried language](https://en.wikipedia.org/wiki/Currying), so it's often
useful to think about [partially
applying](https://en.wikipedia.org/wiki/Partial_application) arguments. In
this example, if we partially apply the `'a => 'b` function in `map`, we are
left with a function from `t('a) => t('b)`. This function looks a lot like
`'a => 'b`, but "lifted" into our functor context.

The order of the arguments to our `map` function doesn't really matter, and
neither does the name of the function - the important parts are the **types**
of the arguments and return value. We could use the following type instead,
and it would still be the the same concept:

```ocaml
let foo = (t('a), 'a => 'b) => t('b);
```

Another thing you'll commonly see in FP is the manipulation of function args,
like using a `flip` function to turn a function `('a, 'b) => 'c` to `('b, 'a)
=> 'c`.

# Aside on names

A very common complaint about learning functional programming is that the
names of things just don't really have much semantic meaning. Many of the
names come from cateogry theory and abstract algebra, and sometimes the names
of the mathematicians who discovered or developed the concepts. I've seen
lots of posts and comments on the internet about how we should call this
"functor thing" `Mappable` rather than `Functor`, because `Mappable` has a
more intuitive and semantic meaning. This is a fair complaint and suggestion,
but it starts to break down a bit when we start to get into more abstract and
powerful concepts.

The most important aspect of a name is that it gives us a way to refer to a
concept in its entirety using one or two distinct words. The same argument
for a common language was made in the venerable OO [Design
Patterns](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612/ref=sr_1_3?keywords=design+patterns&qid=1572666058&sr=8-3)
book, [Domain-Driven
Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215/ref=asc_df_0321125215/?tag=hyprod-20&linkCode=df0&hvadid=312118197030&hvpos=1o1&hvnetw=g&hvrand=11366912520175513035&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=9028890&hvtargid=pla-449269547899&psc=1),
and just about every branch of learning from mathematics and science to law
and history.

There was once a time where the word "Integer" probably didn't mean anything
to you, but as you learned about the definition of an integers, and what you
can do with them, the abstract name became associated with the concrete
concept, and now we use the term `int` without a second thought, knowing
exactly what it is. I would go so far to say that something that was once
completely unknown is now viewed by most as simple and obvious. Someone might
have proposed that we call these things "Addables" or "Operatables" or
whatever to give it a more semantic name, but ultimately, once you learn what
the name denotes, the name really doesn't end up mattering anymore, and it's
often more difficult to capture the full semantics in a single semantic name
anyway.

This is not to say that names don't matter, but it's quite common in FP for
concepts to have abstract names, and some functions to have opaque operators
which will require some learning and patience. It's something that you'll
have to get used to.

# Functor laws

We now know that a functor deals with a type `t('a)` and has a function `map`:

```ocaml
type t('a);
let map: ('a => 'b, t('a)) => t('b);
```

There are two additional things that we must satisfy in order for something to
be considered a valid functor: the "functor laws."

## First functor law - structure preserving map

The first law requires that the `map` function cannot change the structure of
the data type - it can only change the values contained within the functor
using the `'a => 'b` function. This is sometimes called a
"structure-preserving" map operation.

The first functor law can be written like this:

```ocaml
// The identity function - simply returns the value given as the argument
let id = a => a;

// The === here just means these things must be the same
map(id, fa) === fa;
```

Basically if you map the identity function over your functor, you should get
back your functor completely unchanged, both in content and structure. In
concrete terms, the `map` function for `list('a)` (or any other functor) is
not allowed to shuffle things around, copy values, lengthen (create new
structure), nor shorten (remove structure) the list, etc.

This concept is important, because if you have a valid functor, you can be
confident that the `map` operation will only apply the given function to each
value contained within your functor, and will not change the structure of your
data type.

## Second functor law - composition

The second functor law has to do with function composition. Basically, if you have
a functor `t('a)`, and two functions that you'd like to map:

```ocaml
let fa: t('a) = ...;

let aToB: 'a => 'b = ...;

let bToC: 'b => 'c = ...;
```

We can create two specialized map functions by [partially
applying](https://en.wikipedia.org/wiki/Partial_application) our `aToB` and
`bToC` functions to `map`:

```ocaml
let mapAToB: t('a) => t('b) = map(aToB);

let mapBToC: t('b) => t('c) = map(bToC);
```

This is an example of "lifting" our pure `'a => 'b` functions into the
functor context `t('a) => t('b)`.

Now let's [compose](https://en.wikipedia.org/wiki/Function_composition) these
two functions to create a function from `t('a) => t('c)`. First let create a
helper function and operator that we can use for composing functions:

```ocaml
let andThen: ('a => 'b, 'b => 'c) => ('a => 'c) = (aToB, bToC) => a => bToC(aToB(a));

let (>>) = andThen;
```

Now we can create a function from `t('a) => t('c)` by composing our functions `t('a) => t('b)`
and `t('b) => t('c)`:

```ocaml
let mapAToC: t('a) => t('b) = mapAToB >> mapBToC;
```

Ok, so we've composed our two specialized map functions into a specialized map function that
does both map operations.  Now let's try composing our `'a => 'b` and `'b => 'c` functions:

```ocaml
let aToC: 'a => 'c = aToB >> bToC;
```

Now apply that `'a => 'c` function to our map, creating our function `t('a) => t('c)`:

```ocaml
let mapAToC: t('a) => t('c) = map(aToC);
```

We've now created two versions of the function `t('a) => t('c)` - one version
by composing two specialized partial-applications of `map`, and one using a
composed function within `map`. The second law states that these two
specialized `t('a) => t('c)` functions must be equal. In other words (`===`
just means these things are equal):

```ocaml
map(aToB) >> map(bToC) === map(aToB >> bToC)
```

This is probably confusing (and poorly explained), but basically what it
means is that mapping one function `f`, then mapping another function `g`
over a functor in two passes should the same as mapping the composed function
`f >> g` over the functor in a single pass. This law is important in that if
we have a valid functor, we know that we can make a pretty substantial
optimization - rather than mapping two different functions over our functor
in two passes, we can do the same thing in a single pass by composing the
functions.

## Why the laws matter

The laws exist to make sure we have a concrete definition as to what makes a
functor a functor. Unfortunately, we can't easily express these laws via the
ReasonML type system, and this is a common problem with many FP languages. A
common solution to this shortfall is to use a [property-based
testing](https://dev.to/jdsteinhauser/intro-to-property-based-testing-2cj8)
library like [bs-jsverify](https://github.com/Risto-Stevcev/bs-jsverify).
This style of testing basically has you setup "properties" of a type, like
the functors laws for example, and then the test framework will test that
property or law against your implementation using a set of generated inputs
to ensure that it passes the law in all cases. I hope to give a more concrete
example of this in a future blog post.

# The `FUNCTOR` typeclass (module type)

Let's acknowledge that the laws are critically important, but we're going to
just file them away for now, and return to more concrete things. Below is how
you might **actually** define a functor as a ReasonML module type (remember
that module types are sort of like interfaces for actual concrete modules):

```ocaml
module type FUNCTOR = {
  type t('a);
  let map: ('a => 'b, t('a)) => t('b);
};
```

I'm going to use all-caps for our module types to differentiate them from the
actual implementation modules. In FP languages like Haskell, PureScript, and
Scala (among others), this interface concept is called a "typeclass" and the
implementation concept is called an "instance of the typeclass" or just
"instance." So we can say that `FUNCTOR` is our "typeclass module type," and
the things we're going to implement below are our "instances" of our
typeclass. You might notice that a typeclass is similar in concept to an OO
interface, and in some ways they are. One key difference is that your
implementation "instance" is a standalone module or value and is not tied to
a another "host" type like it would be in an OO language where your functor
type might do something like `class List extends Functor`. In some ways an
instance might be closer to the OO [adapter
patter](https://en.wikipedia.org/wiki/Adapter_pattern), but still a little
different.

Let's make a few quick observations about this `FUNCTOR` typeclass. In order
to provide an implementation of this "interface," we must:

1. Specify a type with a single type parameter `t('a)`
2. Provide a `map` function that conforms to the given type
3. Implicitly follow the functor laws in our implementation of `map`, as they
   are not guaranteed by the types alone

With this type `t('a)` and this `map` function, let's also notice that a
`FUNCTOR` does not give you any way to put a value of type `'a` into your
functor. It also doesn't know how to construct a value of type `'a`. Finally
it doesn't know how to do other things like flattening a `t(t('a))` into a
`t('a)`, adding structure like turning a `t('a)` into a `t(t('a))`, etc. All
it knows how to do is apply a pure function `'a => 'b` to each value '`a` in
`t('a)` to produce a value of type `t('b)`. (It can't produce values of type
`'b` out of thin air either - it doesn't know or care what types `'a` and
`'b` are!).

# List functor

Now let's implement `FUNCTOR` for `list('a)`. These implementations are purely
for demonstration purposes, so no attempt is made to optimize anything. I'm going
to follow a convention of creating a wrapper module named after my main type, like
`module List`, then implement a few things within this module, then finally implement
my `Functor` as a nested submodule of `List`:

```ocaml
module List = {
  // Alias our type, so we can use `List.t('a)` and `list('a)` interchangably

  type t('a) = list('a);

  // Implement our map function
  // This implementation is recursive, so we're using `let rec`

  let rec map = (f, list) => switch(list) {
    | [] => []
    | [head, ...tail] => [f(head), map(f, tail)]
  };

  // Now let's define our FUNCTOR as a module that implements our module type FUNCTOR

  module Functor: FUNCTOR with type t('a) = t('a) = {
    // Here we just define the members of the FUNCTOR module type
    // We use nonrec here so the compiler knows we're not trying to define
    // a recursive type here, but that the t('a) on the right side refers
    // to the t('a) defined above.

    type nonrec t('a) = t('a);

    let map = map;
  };

  // Another way to define this without using our `type t('a) = list('a)` alias would be:

  module Functor: FUNCTOR with type t('a) = list('a) = {
    type t('a) = list('a);
    let map = map;
  };
};
```

The `module Functor: FUNCTOR` part is saying that we intend for our module
to implement the `FUNCTOR` module type, and we want the compiler to check this
for us. You can leave off the `: FUNCTOR`, and the compiler should be able to
infer this.

The `with type t('a) = list('a)` business is a mechanism for exposing
information about the types contained in our module to the outside world. To
be honest, I'm not an expert on OCaml modules and how they interact with
types, so I won't attempt to explain any of this, but just know that it's
important in this case for the outside world to know that the type `t('a)`
inside our `List.Functor` is the same type as (an alias of) `list('a)`.

Now we have a module `List.Functor` which conforms to the interface defined
by the `FUNCTOR` module type. In other words, `List.Functor` is an instance
of the typeclass `FUNCTOR`. And in OCaml/ReasonML modules are "first-class,"
so we can actually pass this `List.Functor` module around as a value, and use
it in places that want to operate on `FUNCTOR`s.

Also, while we are not attempting to prove that our implementation conforms
to the functor laws above, you can sort of intuitively tell that it conforms
to the first functor law (mapping identity) by just looking. The `'a => 'b`
function is applied to the head value, then the `map` function is called
recursively to apply the function to the tail list. Nothing is being done to
shuffle values around, copy them, etc. and if you were to pass the `id: 'a =>
'a` function to our `map`, it's pretty clear the list would not be modified.
The composition law is a little harder to see intuitively, so we'll just
ignore it for now.

# Aside on learning this

Now that we've seen how to implement a `FUNCTOR` for `list('a)` (and how easy
it is!), I'd recommend trying to implement functor for some of the types
below yourself - I found it was very helpful to learn this stuff by going
through the motions myself, rather than just reading.

# Array functor

The `array('a)` type has a functor, but it's basically exactly the same as
`list('a)`, so I'm going to skip over it. The only real difference is how
`map` is implemented (just because `list('a)` and `array('a)` are different
types), so if you want to give it a shot, go for it!

# Option functor

Now let's implement `FUNCTOR` for `option('a)`. I'm going to follow the convention
of defining a wrapper module for the main type, and aliasing the type `t('a)`, then
defining the `Functor` as a nested submodule of our main module.

```ocaml
module Option = {
  type t('a) = option('a);

  let map = (f, option) => switch(option) {
    | Some(a) => Some(f(a))
    | None => None
  };

  module Functor: FUNCTOR with type t('a) = t('a) = {
    type nonrec t('a) = t('a); // alias above type
    let map = map; // alias above map function
  };
};
```

An `option('a)` is basically a list of 0 or 1 items, so the implementation is
super easy. Once again, the first functor law can be seen intuitively.

Hopefully at this point, if functors were scary before, they are very un-scary
now. Maybe functors were never that scary to begin with...

# Result functor

`Belt.Result.t('a, 'e)` is our first example type that doesn't exactly fit
the `t('a)` requirement of functor. We can easily implement `map` as a
function, because we can just ignore the error `'e` case, and only map the
success `'a` case, but the `Functor` module is not quite as straightforward.

Below is the initial cut at implementing `map`:

```ocaml
module Result = {
  type t('a, 'e) = | Ok('a) | Error('e);

  let map = (f, fa) => switch(fa) {
    | Ok(a) => Ok(f(a));
    | Error(e) => Error(e);
  };
};
```

As you can see, in the `map` function, we just ignore the error side, and
only apply the function to our `Ok(a)` value.

Now we want to implement `FUNCTOR`, but we're immediately going to run into a
problem - `FUNCTOR` wants us to give it a `t('a)` - so what do we do with
this error type `'e`?

One way to implement this is to create a very simple `module type` which
simply acts to specify a type, then to use a ReasonML [module
functor](https://v1.realworldocaml.org/v1/en/html/functors.html) (reminder
that "module functors" are more like "module functions" or "module
constructors", and not the map functors we're talking about in this article)
to make our `Result` functor a function of some error type, at the module
level. This is a little funky, especially coming from other non-module-based
languages, but hopefully it makes sense in a simple example:

```ocaml
// Create a super simple module type `TYPE` which just acts to capture a type `t`
// This can live outside our `Result` type:

module type TYPE = {
  type t;
};

// Now we can setup a module functor to allow us to construct a `FUNCTOR`
// for a given error type, specified by a TYPE module:

module Result = {
  type t('a, 'e) = | Ok('a) | Error('e);

  let map = (f, result) => switch(result) {
    | Ok(a) => Ok(f(a));
    | Error(e) => Error(e);
  };

  module MakeFunctor = (E: TYPE) => {
    // Here we reference our above Result.t, but with the error type "locked-in"
    // to the type given to us by `E` - which is a module conforming to the module type `TYPE`

    type t('a) = t('a, E.t);

    // The map function just works regardless of `'e`/`E` - we needed the `E` to
    // satisfy the `type t('a)` type of `FUNCTOR`. In ReasonML/OCaml, functions
    // tend to be "more powerful/polymorphic" than modules in terms of dealing
    // with types, so we tend to only have to do module functor gymnastics to
    // satisfy module types, and not so much for functions.

    let map = map;
  }
```

In order to actually construct a `Functor` module for `Result`, we need to use
the `Result.MakeFunctor` module functor (the functor functor!), like below:

```ocaml
module Error: TYPE = {
  type t = | Error1 | Error2; // can be any type
};

module ResultFunctor = Result.MakeFunctor(Error);
```

You can also inline the `TYPE` like this:

```ocaml
module ResultFunctor = Result.MakeFunctor({ type t = string });
```

However, you unfortunately can't do things like this:

```ocaml
Result.MakeFunctor({ type t = string }).map(...);
```

At this point you might be wondering why you'd even bother creating this
`MakeFunctor` thing and requiring the `TYPE` thing having to construct a
special `ResultFunctor`, when it seems you can just use `Result.map`
directly. That's a good and valid point, but this post is just trying to set
you up for some more powerful features that become available to you by
creating instances of typeclasses. The point of typeclasses is that they
provide you with a higher-level of abstraction, and provide ways to create
more abstractions on top of them. In a future post, we'll talk about
applicatives and monads, and then hopefully the power, abstraction, and
purpose of this will become more clear.

# Tree functor

# Function functor

# Function that's not a functor

# Parser/decoder functor

# RemoteData-like functor

# Custom data type functor

# Future functor

# What can you do with a `FUNCTOR`?

# `FUNCTOR` extensions and operators

# Using `FUNCTOR` as a first-class module