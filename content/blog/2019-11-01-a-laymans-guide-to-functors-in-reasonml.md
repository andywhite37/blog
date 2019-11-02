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

# The FUNCTOR module type

Let's acknowledge that the laws are critically important, but we're going to
just file them away for now, and return to planet earth. Here is how you
might **actually** define a functor as a ReasonML module type:

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
"instance." So we can say that `FUNCTOR` is our "typeclass module type." You
might notice that a typeclass is similar in concept to an OO interface, and
it is in some ways. One key difference is that your implementation "instance"
is a standalone module or value and is not tied to a another "host" type like
it would be in an OO language where your functor type might `extends
Serializable` or whatever.

A few quick observations about this `FUNCTOR` typeclass - to provide an implementation
of this "interface," we must:

1. Specify a type with a single type parameter `t('a)`
2. Provide a `map` function that conforms to the given type
3. And we must implicitly follow the functor laws in our implementation of
`map`, as they are not guaranteed by the types alone

With this type `t('a)` and this `map` function, let's also notice that a
`FUNCTOR` does not give you any way to put a value of type `'a` into your
functor. It also doesn't know how to construct a value of type `'a`. Finally
it doesn't know how to do other things like flattening a `t(t('a))` into a
`t('a)`. All it knows how to do is map a pure function `'a => 'b` over your
type `t('a)` to produce a value of type `t('b)`. (It can't produce values of
type `'b` out of thin air either).

# List functor

Now let's implement `FUNCTOR` for `list('a)`. These implementations are purely
for demonstration purposes, so no attempt is made to optimize anything:

```ocaml
module List = {
  // Let's just alias our type t('a) to list('a)

  type t('a) = list('a);

  // This implementation is recursive, so we're using `let rec`

  let rec map = (f, list) => switch(list) {
    | [] => []
    | [head, ...tail] => [f(head), map(f, tail)]
  };

  // Now let's define our FUNCTOR as a module:

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
to the functor laws above, you can kind of intuitively tell that it conforms
to the first functor law of identity by just looking. The function is applied
to the head value, then the function is called recursively to apply the
function to the tail list. Nothing is being done to shuffle values around,
copy them, etc. and if you were to pass the `id: 'a => 'a` function to our
`map`, it's pretty clear the list would not be modified. The composition law
is a little harder to see intuitively, so we'll just ignore it for now.

# Aside on learning this

Now that we've seen how to implement a `FUNCTOR` and how simple it is, I'd
recommend trying to implement functor for some of the types below yourself -
I found it was very helpful to learn this stuff by going through the motions
myself, rather than just reading.

# Option functor

Now let's implement `FUNCTOR` for `option('a)`. I'm going to follow the convention
of defining a wrapper module for the main type, and aliasing the type `t('a)`, then
defining the `Functor` as a nested "submodule" of our main module.

```ocaml
module Option = {
  type t('a) = option('a);;

  // I like to implement the typeclass function as a top-level members of a module, and
  // then just alias them insde the typeclass instance modules. The type system in OCaml
  // is more powerful than the module system.

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

Hopefully at this point, if functors were scary before, they are very unscary
now. Maybe functors were never scary...

# Result functor

`Belt.Result.t('a, 'e)` is our first example that doesn't exactly fit the `t('a)`
requirement of functor. We can easily implement `map` as a function, because we
can just ignore the error `'e` case, and only map the success `'a` case, but the
`Functor` module is not quite as straightforward.

...

# Function functor

# Function that's not a functor

# Parser/decoder functor

# Future functor

# Functor extensions

# First-class modules