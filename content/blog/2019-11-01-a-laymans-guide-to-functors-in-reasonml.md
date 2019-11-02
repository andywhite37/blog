---
title: "A Layman's Guide to Functors in ReasonML"
date: 2019-11-01T17:50:46-06:00
draft: false
---

**This is a WIP**

In [my previous post](/blog/2019-10-31-hello-world) I talked about my
background and how I started on my journey to learn typed functional
programming. I'll again preface these posts with a note that I don't have a
background in category theory, so these posts are intended to help introduce
things from a boots-on-the-ground perspective. Please feel free to correct me
on any points I mess up.

# Functors

In this post, I would like to talk about functors - not [OCaml/ReasonML
module functors](https://v1.realworldocaml.org/v1/en/html/functors.html), but
functors as they relate to the `map` function. I would wager that most
software developers have probably used a `map` function for lists or arrays,
and probably used a map-like function when dealing with asynchronous types
like `Future` or `Promise`, to convert a value of some type `A` to a value of
some other type `B`. If you're lucky, you've used `map` on a type like
`Option`, `Maybe`, `Either` or `Result`. If you're really blessed, maybe
you've used it in something like a JSON decoder function to transform a
decoded number into some other type.

# ReasonML - a quick aside

I'm going to use the language [ReasonML](https://reasonml.github.io) for
these posts, not because it's the most powerful language around (it's not),
but because it provides an interesting platform on which to build up some
examples. If you're not familiar with ReasonML, it's basically an alternative
syntax for the programming language [OCaml](https://ocaml.org). The ReasonML
programming language can be losslessly translated to OCaml and vice-versa,
because the two langauges share the same [abstract syntax tree
(AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree) - the data
structure that represents the parsed code.

One of the reasons ReasonML was created was to provide a syntax that would be
more familiar to developers coming from JavaScript/ECMAScript. ReasonML has a
few different compilers which are able to produce JavaScript as a "backend,"
meaning that it can transpile ReasonML (or OCaml) code to JavaScript, for use
in the browser, or in Node.js, for example. The compiler I most use at the
moment is called [BuckleScript](https://bucklescript.github.io/en/) - which
is actually an OCaml to JS compiler. To make it work with ReasonML, you use a
separate tool to translate the ReasonML code to OCaml code before compiling
it with the BuckleScript compiler to produce the JavaScript output, which you
then have to bundle with a tool like [Webpack
:scream:](https://webpack.js.org), [Parcel](https://parceljs.org),
[Rollup](https://rollupjs.org/guide/en/) or whatever the latest
*so hot right now* bundler happens to be in the JavaScript community.

The workflow is basically:

1. Write ReasonML code
2. Convert ReasonML code to OCaml
3. Compile OCaml code with Bucklescript to produce JavaScript code
4. Bundle JavaScript code to produce something you can use on the web or in Node.js

If this sounds a little crazy to you, you're right, it is a little crazy.
However steps 2 and 3 kind of happen at the same time by the same tool, and
the BuckleScript compiler is blazingly fast, so if you're writing ReasonML, you
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
out if you are interested.

# Back to functors

To jump right in, I'm going to introduce one way to represent a functor in
ReasonML and then go from there. In ReasonML, we have the concept of
[modules](https://ocaml.org/learn/tutorials/modules.html), which is a
fundamental and first-class structure in the language. You can sort of think
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

The next aspect of a functor is that it must support a `map` function with
the following type:

```ocaml
let map = ('a => 'b, t('a)) => t('b);
```

Basically, `map` is a function that accepts an `'a => 'b` function to convert
a value of type `'a` to a value of type `'b`, a value of type `t('a)`, and
returns to you a value of type `t('b)`.

Another way to look at `map` is that it takes a function of type `'a => 'b`
and returns to you a function of type `t('a) => t('b)`.

The order of the arguments doesn't really matter, and neither does the name
of the function - the important part is the types of the arguments and return
value. We could use this type instead, and it's the same thing:

```ocaml
let foo = (t('a), 'a => 'b) => t('b);
```

# Aside on names

A very common complaint about learning functional programming is that the
names of things just don't really have much semantic meaning at all. Many of
the names come from cateogry theory and abstract algebra, and sometimes the
mathemeticians who discovered or developed the concepts. I've seen lots of
posts and comments on the internet about how we should call this thing
`Mappable` rather than `Functor`, because `Mappable` has a more intuitive and
semantic meaning. This is a fair complaint, but it starts to break down a bit
when we start to get into more abstract and powerful concepts.

The most important aspect of a name is that it gives us a way to refer to a
concept in its entirety using one or two distinct words. The same argument
for a common language was made in the well-respected OO [Design
Patterns](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612/ref=sr_1_3?keywords=design+patterns&qid=1572666058&sr=8-3)
book.

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
be considered a valid functor - the "functor laws."

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
concrete terms, the `list('a)` `map` function is not allowed to shuffle
things around, copy values, lengthen, nor shorten the list, etc.

This concept is important, because if you have a valid functor, you can be
confident that the `map` operation will only apply the given function to each
value contained in your functor, and will not change the structure of your
data type.

## Second functor law - composition

The second functor law has to do with function composition. Basically, if you have
a functor `t('a)`, and two functions that you'd like to map:

```ocaml
// Our input functor "fa" (functor of a)
let fa: t('a) = ...;

// Our two functions that we want to map over our functor
let aToB: 'a => 'b = ...;
let bToC: 'b => 'c = ...;

// Our functor map function
let map: ('a => 'b, t('a)) => t('b) = ...;
```

Let's now create two specialized map functions by [partially
applying](https://en.wikipedia.org/wiki/Partial_application) our `aToB` and
`bToC` functions to `map`:

```ocaml
// Create our specialized map functions by locking in the mapping functions, and leaving
// the functor argument `t('a)` open.

let mapAToB: t('a) => t('b) = map(aToB);
let mapBToC: t('b) => t('c) = map(bToC);
```

This is an example of "lifting" our pure `'a => 'b` functions into the functor context `t('a) => t('b)`.

Now let's compose these two functions to create a function from `t('a) => t('c)`:

```ocaml
// andThen is our function composition function
// Takes a function 'a => b and a function 'b => 'c and returns a function 'a => 'c

let andThen: ('a => 'b, 'b => 'c) => ('a => 'c) = (aToB, bToC) => a => bToC(aToB(a));

// Compose our two specialized `mapAToB` and `mapBToC` functions to 

let mapAToC: t('a) => t('b) = andThen(mapAToB, mapBToC);

// You can also write this like:

let mapAToC: t('a) => t('b) = fa => fa |> mapAToB |> mapBToC;
```

Ok, so we've composed our two specialed map functions into a specialized map function that
does both map operations.  Now let's try composing our `'a => 'b` and `'b => 'c` functions:

```ocaml
// Compose our two pure functions to create a function 'a => 'c

let aToC: 'a => 'c = andThen(aToB, bToC);

// Now we can create a specialized mapAToC function using this:

let mapAToC: t('a) => t('c) = map(aToC);
```

The second law states that these two specialized `mapAToC` functions must be equal.
In other words:

```ocaml
// map from 'a to 'c by Composing the specialized map functions

let mapAToC_1: t('a) => t('c) = andThen(map(aToB), map(bToC));

// map from 'a to 'c by mapping a composed map function

let mapAToC_2: t('a) => t('c) = map(andThen(aToB, bToC));

mapAToC_1 === mapAToC_2
```

This is probably confusing (and poorly explained), but basically what it
means is that mapping one function `f`, then mapping another function `g`
over a functor should the same as mapping the composed function `andThen(f,
g)` over the functor in a single pass. This law is important in that if we
have a valid functor, we know that we can make a pretty substantial
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
to ensure that it passes the law in all cases.

# Finally, the FUNCTOR module type