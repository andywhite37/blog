--- 
date: 2019-11-07T22:28:54-07:00
title: "A Layman's Guide to Applicatives in ReasonML"
#series: ["A Layman's Guide to Functional Programming"]
categories: ["Software Development", "Functional Programming"]
tags: ["ReasonML", "OCaml", "Functors", "Layman's Guide"]
draft: true
---

# Other posts in this series:

- [A Layman's Guide to Functors in ReasonML](2019-11-01-a-laymans-guide-to-functors-in-reasonml)

# Applicative functors

I'll start this post off with a tantalizing quote that I first heard uttered 
by [Kris Nuttycombe](https://twitter.com/nuttycom):

> In functional programming, applicatives are the essence of parallel
processing, and monads are the essence of sequential processing.

I'm not sure if that's the exact quote, but I think it captures the idea. In
this post about applicative functors, and my next planned post about monads,
I hope to provide some intuitions about this quote.