---
title: "Projects"
date: 2019-11-01T13:31:20-06:00
draft: false
---

# ReasonML

My colleague [Michael Martin-Smucker](https://github.com/mlms13) and I develop and
maintain a small but growing ecosystem of libraries for ReasonML/BuckleScript
in the [Reazen](https://github.com/reazen) GitHub org.

- [relude](https://github.com/reazen/relude)
  - relude is our "standard library replacement" for ReasonML/BuckleScript.
  - The purpose of this library is to provide a "batteries included" style of
    prelude/stdlib based on the powerful abstractions from category theory and abstract algebra.
  - The library was primarily inspired by the ecosystems of Haskell,
    PureScript and Scala libraries like scalaz, cats, and shapeless
  - In addition to the math-based abstractions, we provide all the other
    helper modules and functions that you'd expect from a reasonable standard
    library.
- [relude-parse](https://github.com/reazen/relude-parse)
  - relude-parse is a monadic parsing library, inspired by all the amazing prior art
    in Haskell/PureScript/Scala
- [relude-reason-react](https://github.com/reazen/relude-reason-react)
  - A "principled" set of helper utilities for [ReasonReact], based on relude's abstractions
- [relude-fetch](https://github.com/reazen/relude-fetch)
  - A interop library for the `fetch` API, using the pure abstractions provided by relude
- [relude-eon](https://github.com/reazen/relude-eon)
  - A date/time library for ReasonML
- [relude-csv](https://github.com/reazen/relude-csv)
  - A pure functional CSV parsing library based on relude-parse
- [relude-url](https://github.com/reazen/relude-url)
  - A pure functional URL parsing library based on relude-parse

# NixOS

My co-worker [Jeff Simpson](https://github.com/fooblahblah) convinced me to
try out NixOS when I had recently gotten a new work computer. I still don't
completely know what I'm doing, but I'm enjoying using this quite different
and innovative Linux distro.

# Scala

I don't have any open-source work to list in the Scala ecosystem, but I've
had the opportunity to hone my pure functional programming skills using
libraries like cats, shapeless, atto, circe, and ZIO.

# PureScript

I haven't done as much PureScript as I would like to, but I did create a library
for purescript-halogen to wrap a now-deprecated Material UI library (material design lite/mdl):

- [purescript-halogen-mdl](https://github.com/andywhite37/purescript-halogen-mdl)

# Haxe

- [haxpression](https://github.com/andywhite37/haxpression) and [haxpression2](https://github.com/andywhite37/haxpression2)
  - Math expression parser and evaluation libraries for Haxe
  - v1 was a port of a JavaScript library, and v2 was a complete rewrite using a monadic parser approach
- [graphx](https://github.com/andywhite37/graphx)
  - A graph traversal/utility library for Haxe
- [thx.core](https://github.com/fponticelli/thx.core)
  - I had the good fortune of working for [Franco Ponticelli](https://github.com/fponticelli), who
    is the creator and maintainer of the `thx` family of libraries for Haxe.
  - I contributed features and bug fixes for the main `thx.core` library and several
    of the other libraries in the ecosystem
- [abe](https://github.com/abedev/abe)
  - I worked with the `abe` library (a Haxe binding to the express.js JavaScript library), and
    the associated set of bindings to other js-based `npm` packages.
- [doom](https://github.com/fponticelli/doom)
  - Worked with and on a innovative homegrown virtual DOM UI library for Haxe

# Unity

I don't have much to show beyond some tutorial code, but at one time I
decided it would be fun to learn the Unity game development framework. I
enjoyed using the tools, but ultimately, I kind of fizzled out having to use
C# again, after having gotten used to functional programming idioms.