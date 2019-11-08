---
date: 2019-11-01T17:50:46-06:00
title: "A Layman's Guide to Functors in ReasonML"
#series: ["A Layman's Guide to Functional Programming"]
categories: ["Software Development", "Functional Programming"]
tags: ["ReasonML", "OCaml", "Functors", "Layman's Guide"]
draft: false
---

In [my intro post](/posts/2019-10-31-hello-world) I talked about my background
and how I started on my journey to learn typed functional programming. I'll
again preface these posts with a note that I don't have a background in
category theory, so these posts are intended to help introduce things from a
boots-on-the-ground perspective. Please feel free to correct me on any points
I've messed up. I'm also not an OCaml expert, so there may be techniques or
coding conventions here that are not completely correct. The purpose of these
blogs is not to achieve rigorous perfection but to help impart some intuition
on the topics. If you find any issues with code examples, please let me know,
and I'll fix them.

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
typed pure functional programming.

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
whatever to give it a more semantic (though insufficient) name, but
ultimately, once you learn what the name denotes, the name really doesn't end
up mattering anymore, and it's often more difficult to capture the full
semantics in a single semantic name anyway.

This is not to say that names don't matter, but it's quite common in FP for
concepts to have abstract names, and for some functions to have opaque
operators. This will require some learning and patience, and it's just
something you'll have to get used to. That said, I've experienced the
firehose of unknown words, so I'm not trying to brush this off, but just
trying to set expectations. Just remember, you don't have to understand all
of this at once! Just to try build up, cement, and apply your knowledge over
time, and it will grow.

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

// The == here just means these things must be the structurally the same
map(id, fa) == fa; // true
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
specialized `t('a) => t('c)` functions must be equal. In other words (`==`
just means these things are structurally equal):

```ocaml
map(aToB) >> map(bToC) == map(aToB >> bToC)
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

# The FUNCTOR typeclass (module type)

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
    | [head, ...tail] => [f(head), ...map(f, tail)]
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

Now let's try another data type: a binary tree. Using recursion with `map`
makes this very similar to `list('a)`, so I'll just jump into the example:

```ocaml
module Tree = {
  type t('a) = | Leaf | Node(t('a), 'a, t('a));

  let rec map = (f, tree) => switch(tree) {
    | Leaf => Leaf
    | Node(left, value, right) => Node(map(f, left), f(value), map(f, right))
  };

  module Functor: FUNCTOR with type t('a) = t('a) = {
    type nonrec t('a) = t('a);
    let map = map;
  };
};
```

Our `Tree` is a variant that's either a `Leaf` (empty tree), or it's `Node`
which has a value of type `'a`, and a sub-tree on the left side, and a
sub-tree on the right side. A tree with one value would be `Node(Leaf,
myValue, Leaf)`.

The `map` function just recurses down the left and right branches of the
tree, and the value at each node is updated with `f`. As you can see we're
not modifying the structure of the tree - just visiting (and updating) the
value at each node with the pure function `'a => 'b`.

# RemoteData functor

We've now seen functors for some commonly-used, general data types, but you
can also define functors for more domain-specific types. Let's look at the
Elm
[RemoteData](https://package.elm-lang.org/packages/krisajenkins/remotedata/latest/)
type, which also exists as a ReasonML library
[bs-remotedata](https://github.com/FabienHenon/bs-remotedata).

The type in question basically looks like the following:

```ocaml
module RemoteData = {
  type t('a, 'e) =
    | NotAsked
    | Loading
    | Failure('e)
    | Success('a);
};
```

This type is a variant that's used to represent the different states in which
an asynchronous data fetch operation might exist. E.g. the `NotAsked` state
is typically used as the initial state where there's no data, and a request
has not yet been made. Before the async `fetch` request is made, the state
might be updated to `Loading` to indicate that there is work happening. When data
comes in, the state is set to `Success('a)`, where `'a` is the data value that
came in, and if the request fails, the state is set to `Failure('e)` where `'e`
is the custom error type, which might be a string, or a more specific variant or
record type.

The type is similar to `Result.t('a, 'e)` in that it has two type parameters,
so the functor implementation is going to look very similar. The `NotAsked`
and `Loading` constructors carry no data, so they will just pass through
untouched in the `map` function. Our map function operates on the
`Success('a)` channel, so the `Failure('e)` value will also get passed
through untouched:

```ocaml
module RemoteData = {
  type t('a, 'e) =
    | Init
    | Loading
    | Failure('e)
    | Success('a);

  let map = (f, remoteData) => switch(remoteData) {
    | Init => Init
    | Loading => Loading
    | Failure(e) => Failure(e)
    | Success(a) => Success(f(a))
  };
};
```

To define the `FUNCTOR` module, we have to use the module functor trick like we did for
`Result` to lock in the error type. This time I'll show another technique to do this:

```ocaml
module RemoteData = {
  type t('a, 'e) = ...
  let map = ...

  module WithError = (E: TYPE) => {
    module Functor: FUNCTOR with type t('a) = t('a, E.t) = {
      type t('a) = t('a, E.t);
      let map = map;
    };

    // You can use E and E.t for more things in here if you need to...
  };
};
```

This time, I'm using a `WithError` module functor that locks in the error
type in `E.t` and then we define the `Functor` module within the `WithError`
module. Doing it this way allows you to use the `E` error module multiple
times within the same scope without having to functorize every module that
needs to use `E`. E.g. when we start to add more typeclass instances, it's
handy to just "functorize" once `WithError`, and then just define all the
other instances normally with `E.t`. Beware that module functors are not
without their costs in ocaml (in module purity, dead code elimination, etc.),
so sometimes it's better to use more smaller functors, rather than fewer
larger functors.

# Custom data type functor

As we've seen with `RemoteData.t('a, 'e)` you can define functors for your
own domain types, not just the common data types like `list('a)`,
`option('a)`, etc. As you do more functional programming, you'll also start
to see functors used in other new and exciting places, like free monads, etc.

When you're making your own polymorphic types, it's worth thinking about
whether a `FUNCTOR` (i.e. a `map` function) might make sense for your data
type. One clue is if your data type has one type parameter that sort of
serves as the "main" or "success" value of the type, and the other type
params maybe serve as supporting types. Another clue is whether your type is
a data structure that carries some single type of data - in this case your
type is almost certainly a functor.

You might also find that your data type has multiple values that make sense
to map, and in these cases you might have a **bifunctor**, **trifunctor**,
**quadfunctor**, etc. These are all basically just types that allow you to
map more than one of the polymorphic types, assuming you can still abide by
the functor laws. The rule of thumb with FP is that someone has likely
already figured it out, named it, and fully-implemented it, so if you think
you've found something interesting, look for prior art.

# Function functor

Now let's look at something that's not just a plain old static data type: a
function! Let's consider the function `'x => 'a` where `'x` is some type we
don't really care about, and `'a` is the type we care about (sort of like in
`Result.t('a, 'e)` where we cared about the `'a`, and not so much about the
`'e` for the purposes of our functor).

Recall the type of `map`:

```ocaml
let map: ('a => 'b, t('a)) => t('b);
```

For our function, we can define our type like this:

```ocaml
module Function = {
  type t('x, 'a) = 'x => 'a;
};
```

Now let's try implementing map:

```ocaml
module Function = {
  type t('x, 'a) = 'x => 'a;

  // ('a => 'b, t('a)) => t('b)
  let map: ('a => 'b, 'x => 'a) => ('x => 'b) = (aToB, xToA) => {
    // We need to return a function of type 'x => 'b, so we know we have an argument
    // of type 'x, and a function from 'x => 'a, and a function from 'a => 'b, so we
    // just call those functions to convert x -> a -> b:

    x => aToB(xToA(x));
  };
};
```

If you recall above in the functor laws section, we implemented a function
for composing function `andThen` which looked at lot like this, and there is
also a function called `compose` which does the same thing, with the
arguments flipped.

```ocaml
let andThen: ('a => 'b, 'b => 'c) => ('a => 'c) = (aToB, bToC) => {
  a => bToC(aToB(a))
};

let (>>) = andThen;

let compose: ('b => 'c, 'a => 'b) => ('a => 'c) = (bToC, aToB) => {
  a => bToC(aToB(a))
};

let (<<) = compose;
```

It turns out the functor for the function `'x => 'a` is just the same thing
as function composition!

Now to implement our actual `Functor` module, we again have to use a module functor
because we're dealing with a type `t('x, 'a)` which has more than one type parameter.
I'll use the `With*` module functor trick again, but you could also do this in
the `module MakeFunctor = (E: TYPE) => { type t('a) = ... }` style.

```ocaml
module Function = {
  type t('x, 'a) = 'x => 'a;

  let map = (aToB, xToA) => x => aToB(xToA(x));

  module WithArgument = (X: TYPE) => {
    module Functor: FUNCTOR with type t('a) = t(X.t, 'a) = {
      type t('a) = t(X.t, 'a);
      let map = map;
    };
  };
};
```

If this seems a little silly to you, it is - it's stupid simple, but it turns
out that the function `'x => 'a` is actually a very powerful and useful
construct, especially when we get to the topic of monads. If you want to read
ahead, search for [reader
monad](https://www.google.com/search?q=reader+monad).

# Function that's not a (covariant) functor

We just looked at the function `'x => 'a`, so what about the function `'a =>
'x`? Let's quickly see if we can implement map for this type:

```ocaml
module Function = {
  type t('a, 'x) = 'a => 'x;

  let map: ('a => 'b, 'a => 'x) => ('b => 'x) = (aToB, aToX) => {
    // Need to return a function 'b => 'x
    b => ??? huh ???
  };
};
```

It turns out we can't implement our `FUNCTOR` for this type. I won't get into
this for now, and hope to in a future blog post, but the reason for this has
to do with [covariance vs. contravariance](https://en.wikipedia.org/wiki/Functor#Covariance_and_contravariance).
The `FUNCTOR` we defined above with the `let map: ('a => 'b, t('a)) => t('b)` function
is called a **covariant** functor, but the functor that we need for the type `'a => 'x`
is called a **contravariant** functor, which looks like this:

```ocaml
module type CONTRAVARIANT = {
  type t('a);
  let contramap: ('b => 'a, t('a)) => t('b); // sometimes called cmap
};
```

If you try to implement `CONTRAVARIANT` for types like `option('a)` you're
going to get confused, just like we got confused trying to implement our
covariant `FUNCTOR` for `'a => 'x`. Try implementing `CONTRAVARIANT` for `'a
=> 'x` though, and think about function composition again.

# Parser/decoder functor

So now we've seen `FUNCTOR` for `'x => 'a` (and mentioned `CONTRAVARIANT` for
`'a => 'x`, but we won't mention `CONTRAVARIANT` again in this article), so
let's see a real-world example of a covariant `FUNCTOR` for a function.

What about a function that takes a `Js.Json.t` value, and "decodes" it into
a type like `Result.t('a, Error.t)`

```ocaml
module Decoder = {
  module Error = {
    type t = | ExpectedString | Other;
  };

  type t('a) = | Decode(Js.Json.t => Result.t('a, Error.t));
};
```

Here our `Decoder.t('a)` is a data type that wraps a function `Js.Json.t =>
Result.t('a, Error.t))`. I'll fix the error type for simplicity, but you could
use a polymorphic error if you want (by using the module functor `TYPE` trick).

If you squint, the function `Js.Json.t => Result.t('a, Error.t))` looks a lot
like the function `'x => 'a`, where `'x` is the type we don't really care
about in terms of mapping (the `Js.Json.t` value), and our `'a` is just
buried in a `Result`.

Let's try implementing `map`:

```ocaml
module Decoder = {
  module Error = {
    type t = | ExpectedString | Other;
  };

  type t('a) = | Decode(Js.Json.t => Result.t('a, Error.t));

  // Our map function needs to return a new decoder `Decode(Js.Json.t => Result.t('b, Error.t))`:

  let map: ('a => 'b, t('a)) => t('b) = (aToB, Decode(jsonToResultA)) => {
    Decode(json => jsonToResultA(json) |> Result.map(aToB));
  };
};
```

Here we're returning a new decode function wrapped in our `Decode`
constructor. The new function accepts a `Js.Json.t` argument, and calls our
original decode function to get the `Result.t('a, 'e)`, then we use the
`Result.map` function to map the `'a` value in the `Result` to the `'b` value
that we want in the end. Since we're dealing with functions here, nothing has
actually done anything yet - no decoders have been run - we are just left
with a new function that accepts a `Js.Json.t` and now gives us a
`Result.t('b, Error.t)`. Stuff like this is where we start to see the real
power of functional programming - just composing pure functions to describe
how to do the work, then doing the work separately.

As a side note, I didn't have to use the `Decode` wrapper for out `t('a)` above -
you can just define your type as just an alias for a function like this:

```ocaml
type t('a) = Js.Json.t => Result.t('a, Error.t);
```

But it's often useful to wrap functions in some type of "container" or
"context" and it also helps to envision our decoder as a more opaque
"context", rather than just a loose function. You can however do all this
without the wrapper context.

Let's see how this works in reality. We'll implement a boolean decoder, then
map the value to a string:

```ocaml
module Decoder = {
  module Error = {
    type t =
      | ExpectedBool(Js.Json.t)
      | OtherError;
  };

  type t('a) =
    | Decode(Js.Json.t => Result.t('a, Error.t));

  let map: ('a => 'b, t('a)) => t('b) =
    (aToB, Decode(jsonToResultA)) =>
      Decode(json => jsonToResultA(json) |> Result.map(aToB));

  let run: (Js.Json.t, t('a)) => Result.t('a, Error.t) =
    (json, Decode(f)) => f(json);

  let boolean: t(bool) =
    Decode(
      json =>
        switch (Js.Json.classify(json)) {
        | JSONTrue => Ok(true)
        | JSONFalse => Ok(false)
        | _ => Error(ExpectedBool(json))
        },
    );

  // Define string, float, int, obj, array, etc. decoders
};

Js.log(
  Decoder.boolean
  |> Decoder.map(v => v ? "it was true" : "it was false")
  |> Decoder.run(Js.Json.boolean(true)),
);

Js.log(
  Decoder.boolean
  |> Decoder.map(v => v ? "it was true" : "it was false")
  |> Decoder.run(Js.Json.boolean(false)),
);

Js.log(
  Decoder.boolean
  |> Decoder.map(v => v ? "it was true" : "it was false")
  |> Decoder.run(Js.Json.string("hi")),
);
```

This logs:

```ocaml
["it was true"]
["it was false"]
[["hi"]]
```

To break this down:

1. We have our type `Decoder.t('a)` which is basically a (wrapped) function
from `Js.Json.t => Result.t('a, Error.t)`
1. I added a `Decoder.run` function. Since our decoder is basically a
function, it makes sense that we'll need to call the function at some point,
but passing in a `Js.Json.t` value to decode. The `run` function basically
just de-structures our decode function and passes the `Js.Json.t` value to it
to produce the `Result.t('a, Error.t)`
    - This is a common pattern in functional programming to build up some
    data structure (which might include functions), and provide a way to
    "run" it. Normally, you build up the structure, and then you can pass
    around and reuse the the structure (because it's pure, and hasn't
    actually done anything yet), and then you just run it when you're ready
    for it to do its thing.
1. I added a `Decoder.boolean` function which is a decoder that succeeds if
the given `Js.Json.t` value is a boolean, and fails if it's not a boolean. 1.
At the bottom, I'm setting up my decoder to expect to parse a boolean (using
`Decoder.boolean`), then I'm mapping my `Decoder` to convert the `bool` to a
`string`, then finally, I'm running the decoder with a few test values
(`true`, `false`, and `"hi"`), to see what happens. The `true` and `false`
cases log the expected strings from the `map`, and the `"hi"` case fails.
    - the `[["hi"]]` is just the JS representation of my
    `ExpectedBool(Js.Json.t)` error.

You can write other decoders or parsers like this, and do simple things like
parsing a single value and mapping it with a `FUNCTOR`, but to decode/parse
structures like JSON object and arrays, you'll want to use more powerful
abstractions like `Applicative` and `Monad`, which I hope to write a blog
about soon.

# Future functor

Let's look at one more real-world example of a `FUNCTOR` - the `Future` type
from [RationalJS/future](https://github.com/RationalJS/future).

If you've done any amount of ReasonML, you've probably had to deal with
`Js.Promise.t('a)`, and I suspect you've not enjoyed it (or maybe you're just
accustomed to the pain if you're coming from JavaScript). I hope to write a
blog in the future about why `Js.Promise.t` is not great in ReasonML, and
what you should use instead, but for now we'll just look at `Future` in terms
of implementing `FUNCTOR`.

`Future` is similar to `Js.Promise.t` in that it's a async "effect" type -
it's a data type that represents a computation that will complete sometime in
the future. The
[Future](https://github.com/RationalJS/future/blob/master/src/Future.re) type
looks like this:

```ocaml
type getFn('a) => ('a => unit) => unit;

type t('a) = | Future(getFn('a));
```

`getFn` might look odd, but all it is is a function that takes a callback `'a
=> unit` as it's only argument - this callback is often called `resolve`.

The `Future` implementation basically creates a function that closes over a
mutable list of "subscribers", and when things subscribe to the `Future`, it
adds those listeners to the array, to be notified when the `Future` is
resolved.

Let's ignore all that and see if we can just implement `map` for `Future`.
Let's pretend that we have a `Future.make` function that takes our resolver, and
would be used like this:

```ocaml
Future.make(resolve => Js.Global.setTimeout(() => resolve("hi"), 40));
```

```ocaml
module Future = {
  type getFn('a) => ('a => unit) => unit;

  type t('a) = | Future(getFn('a));

  let make = ...;

  let map: ('a => 'b, t('a)) => t('b) =
    (aToB, Future(onDoneA)) =>
      make(resolveB => onDoneA(a => resolveB(aToB(a))));
};
```

I've changed the `map` function a little from the real implementation to try
to help with clarity. Basically, the `map` function takes a `Future.t('a)`
and waits for that future to be resolved. We are notified when it resolves
via the `onDoneA` callback. We wrap all this inside a new
`Future.make(resolveB => ...)`. `resolveB` is the function we're supposed to
call when we have a value of type `'b`. So when we get the `'a` from
`onDoneA`, we convert it to `'b` with our `'a => 'b` function and tell the
`Future.t('b)` that we're done by calling `resolveB`.

I'll just leave it at that for now, but you can dig in further by cloning the
[RationalJS/future](https://github.com/RationalJS/future) repo and trying it
out for yourself.

As a side note, this `Future.t('a)` differs from `Js.Promise.t('a)` in that
the `Future.t('a)` cannot fail - it has no way to represent a failed async
computation. To represent a computation that can fail, you should use
`Future.t(Result.t('a, 'e))` - in this case your future value is just a type
that can represent both successful and failed computations.

Compared to `Js.Promise.t`, which hides the error type in some opaque (and
offensive, if you ask me) abstract type, the `Future` approach makes both the
successful value and the error value "first-class" - you have full control
over whether and how your async work can fail.

# Js.Promise functor

For one final example, let's implement `FUNCTOR` for
[Js.Promise](https://reasonml.github.io/docs/en/promise). `Js.Promise` is the
ReasonML binding for the ubiquitous [JavaScript
Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise).

You can implement `map` using the `Js.Promise.then_` function, which is the binding
to the `then` method of JS `Promise`.

```ocaml
module Promise = {
  type t('a) = Js.Promise.t('a);

  let map = (f: 'a => 'b, promiseA: Js.Promise.t('a)): Js.Promise.t('b) =>
    promiseA |> Js.Promise.then_(a => Js.Promise.resolve(f(a)));
  
  module Functor: FUNCTOR with type t('a) = t('a) = {
    type nonrec t('a) = t('a);
    let map = map;
  };
};
```

The usage would look something like this:

```ocaml
Js.Promise.resolve(42)
|> Promise.map(a => a * 2)
|> Js.Promise.then_(b => {
     Js.log(b);
     Js.Promise.resolve();
   });
```

To quickly explain what's going on here, the `then` method of the **native
JavaScript** `Promise` gives you the value from the previous promise, and
allows you to either return a pure value `'b`, a new `Promise` of `'b`,
`null`, nothing at all (aka `undefined`), or throw! This ability to return a
pure value or a new `Promise`, `null`, or `undefined` is not something we can
express directly in the ReasonML type system. Instead, the authors of the
BuckleScript `Js.Promise` binding made it so `Js.Promise.then_` must return a
`Js.Promise`. In our case, we map the value of `'a` to `'b`, and return a
`Promise.resolve(b)`. In vanilla JavaScript, you could just do `a => f(b)`,
but not so in ReasonML.

When we get into monads in a later post, we'll discuss the difference between
returning a pure value, like we do in `map`, and returning a new "monadic"
value, like we're doing here with `Js.Promise.then_`. If you want to read
more about this, a function like `Js.Promise.then_` that lets you return a
new monadic value is often called `bind`, `flatMap`, or `>>=`, and the
function that lets you inject a pure value into your monadic context, like
`Js.Promise.resolve` is often called `pure`, or (confusingly) `return`.
`Js.Promise` is not the greatest thing to use for explaining functors and
monads, because `then` kind of magically does both `map` and `flatMap`, but
it's a familiar example for JavaScript folks.

# What can you do with a FUNCTOR?

There are lots of `FUNCTOR`s in the wild, and you probably use `map` on a
daily basis, but `FUNCTOR` is not the most powerful abstraction in the world
of functional programming. You basically use a `FUNCTOR` when you have some
data structure or context and you want to modify the values inside the
structure/context without affecting the structure itself.

The fact that all a functor knows about is a type `t('a)` and a `map`
function is actually a good thing - it gives you a set of well-defined tools
and constraints, but allows you to abstract over those capabilities, and
provides you with a principled baseline on which to create more powerful
abstractions, like `Applicative`, `Monad` and all sorts of other stuff.

# FUNCTOR extensions and operators

Higher-level constructs like `FUNCTOR` let you implement functions from a
higher-level of abstraction. In other words, rather than implementing
map-related functions for each of `list('a)`, `option('a)`, `Result.t('a,
'e)`, etc., we can implement the function once for `FUNCTOR`, and then all of
the modules we have that have an instance of `FUNCTOR` get those functions
"for free", because these extensions are all implemented just in terms of
`FUNCTOR` (i.e. `t('a)` and `map`).

This is a contrived example, but say you had some data structures like lists,
options, trees, etc., and you needed to convert some data from one type to
another across all these structures. You can setup a `FunctorExtensions`
module functor that lets you define this function once, and can be used by
anything that has a `Functor: FUNCTOR` module.

```ocaml
module FunctorExtensions = (F: FUNCTOR) => {
  let doSomething: F.t('a) => F.t('b) = fa => F.map(someComplexMappingFunction, fa);

  let doAnother: F.t('a) => F.t('b) = fa => F.map(anotherThing, fa);

  // other stuff?
};
```

This is obviously not super compelling, because you can just use `map` on
your types, and pass in the mapping functions without this, but it's just an
example of abstracting on `FUNCTOR`. Also, `FUNCTOR` is again not a super
powerful abstraction, so the benefits of this will be more clear with things
like `Foldable` or `Monad` which I hope to blog about later.

Another use of this `FunctorExtension` technique is to add some common helper
functions and operators to anything that has a `FUNCTOR` instance. There are a few common
functions that other FP languages provide for functors:

```ocaml
module FunctorExtensions = (F: FUNCTOR) => {
  let (<$>) = F.map;

  let (<#>) = (fa, f) => F.map(f, fa);

  let void: F.t('a) => F.t(unit) = fa => F.map(_ => (), fa);

  let voidLeft: (F.t('a), 'b) => F.t('b) => (fa, b) => F.map(_ => b, fa);

  let ($>) = voidLeft;

  let voidRight: ('b, F.t('a)) => F.t('b) => (b, fa) => F.map(_ => b, fa);

  let (<$) = voidRight;

  let flap: (F.t('a => 'b), 'a) => F.t('b) = (fs, a) => F.map(f => f(a), fs);
};
```

`<$>` is the operator version of the `map` function, taken from languages like Haskell.  You use it like this:

```ocaml
let b: option('b) = aToB <$> Some(a);
```

`<#>` is the flipped version of `map`, used like this:

```ocaml
let b: option('b) = Some(a) <#> aToB;
```

`void` is a function that basically replaces all the values in your functor
with the `unit` value `()`. This is the type of thing that might seem odd,
but you often run into situations where you just need to throw away some data
in FP, or convert a type into something that indicates that you don't care
what it is. `void` is a commonly-used name in FP for something that "clears
out" a value by setting it to the `unit` value `()`.

```ocaml
let units: list(unit) = [1, 2 ,3] |> void;
```

`voidLeft` is a function that takes a functor of `'a` and a `'b` value, and
just sticks the `'b` value in the functor regardless of what `'a` value is in
there. `$>` is the operator version of `voidLeft`. You can think of the
operator as doing what looks like - half of a `<$>`/`map`, but it's just
pointing at the value it's going to use to replace all the values of the
functor. `voidLeft` is not a great name for this as it's not voiding like
`void` does, but rather putting a fixed value in, but oh well.

```ocaml
let hi3Times: list(string) = [1, 2 ,3] $> "hi";
```

`voidRight` is the same as `voidLeft` but with the args flipped. `<$` is the
operator version of this - you can think of `<$` similarly to `$>` - the
arguments are just in the opposite order (i.e. "flipped").

```ocaml
let hi3Times: list(string) = "hi" <$ [1, 2 ,3];
```

Finally, `flap` is a strange function that takes a functor of functions and a
single value, and applies those functions to the single value to
"re-populate" the functor with the values produced.

To get access to these extensions, you'd do something like this:

```ocaml
module Option = {
  type t('a) = ...;

  module Functor: FUNCTOR with type t('a) = t('a) = { ... };
  module FunctorExt = FunctorExtensions(Functor);

  // Optionally `include` the extensions directly in your module:
  include FunctorExt;
};
```

With infix operators, it often best to silo them off into their own module,
suitable for use as a local open.

The extensions and infix technique is used extensively in `Relude` - see
[Relude_Extensions_Functor](https://github.com/reazen/relude/blob/master/src/extensions/Relude_Extensions_Functor.re)
and it's use in
[Relude_Option](https://github.com/reazen/relude/blob/master/src/Relude_Option.re)
(both the Extensions and Infix modules), and many other modules.

# Using FUNCTOR as a first-class module

ReasonML/OCaml supports the concept of [first-class
modules](https://v1.realworldocaml.org/v1/en/html/first-class-modules.html),
but I don't believe you can use [higher-kinded
types](https://discuss.ocaml.org/t/higher-kinded-polymorphism/2192) like
`FUNCTOR` in first class modules, so unfortunately, I don't think it's
easy/possible to write a function that expects an arbitrary `FUNCTOR` as an
argument. There are other techniques (like [Lightweight Higher-Kinded
Polymorphism](https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf)
for encoding higher-kinded types in OCaml, but I'm not as familiar with the
techniques described there, so these restrictions might not be true across
the board (or at all!).

As an example, it would be cool if you could do this:

```ocaml
// This doesn't work, or at least I can't get it to work!

let emphasize = (functor: (module FUNCTOR with type t('a) = t(string)), fa: t(string)) => {
  module Functor = (val functor);
  Functor.map(a => a ++ "!", fa);
};

emphasize((module List.Functor), ["hello", "goodbye"]); // ["hello!", "goodbye!"] :pray:
emphasize((module Option.Functor), Some("hello"));      // Some("hello!") :pray:
emphasize((module Result.Functor), Ok("hi"));           // Ok("hi!") :pray:
```

Basically create a function that can work on any `FUNCTOR` without having to
specialize it for each instance, but sadly, I don't think this is possible in
OCaml, because of its lack of higher-kinded types.

This technique does however work for typeclasses that operate on
non-polymorphic types, like `TYPE`, `SHOW` or `EQ` (which will hopefully be
covered in another blog post).

# Conclusion

Well, that was an extremely long blog post, but I hope it will help plant
some seeds for someone who might just be starting their FP journey. I hope to
follow-up this post with a blog about `Applicatives` and then one about
`Monads`, and then go from there. I want to write these longer fundamental
posts so I have something to refer back to when I want to write smaller blogs
about more focused topics in the future.

I hope you enjoyed! If I don't have comments setup in my blog when you read
this, and you have a comment, feel free to open an issue (or pull request)
here: https://github.com/andywhite37/blog.