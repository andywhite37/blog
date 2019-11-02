---
title: "Hello World"
date: 2019-10-31T21:11:53-06:00
draft: false
categories: ["Software Development", "Functional Programming"]
tags: []
---

The world of typed functional programming is a vast, mind-blowing, and often
terrifying place. There are so many things to learn and so many rabbit holes
to go down, it's easy to get overwhelmed, and not know where to even start.
For most of my software development career, I operated in blissful ignorance
of functional programming - I happily wrote object-oriented and imperative
code, mutating data and throwing all sorts of exceptions, and I was actually
pretty content with it. I didn't really see any reason to look into functional
programming at all.

About 5 years ago, I had the good fortune of working with another developer
named [Kris Nuttycombe](https://twitter.com/nuttycom) whom I see as the
person who gave me my first solid kick down the road of functional
programming. He introduced my co-workers and me to the concepts of functors,
applicative validation, monads, semigroups, monoids, and all sorts of other
things. I didn't understand any of it initially, or if I did understand it, I
didn't get why I should care. I resisted, as many of these concepts cast
shade on my precious OO design patterns, but each time I tried to fight and
defend my stance, it just couldn't be done - functional programming (and
math) was just always right, and I was wrong. Every time.

One thing I've observed over the past few years is that learning functional
programming is all about planting seeds and checking back on them later. Kris
would often say something that I didn't understand, and I would have to just
file it away to hopefully encounter it again later. There are all sorts of
monad tutorials out there, but to be honest, maybe 1 out of 20 of them really
clicked for me. It took quite a bit of reading to grasp many of these
concepts, so my purpose for writing this blog is to maybe plant a seed for
someone else. If you're trying to learn functional programming, my advice
would be to cast a wide net, and don't dwell on one resource for too long,
especially if it's not clicking for you. If the "monads are burritos" concept
seems odd or unhelpful to your understanding, move on! Burritos didn't help
me to learn monads, but implementing functor, applicative, and monad
instances for `List`, `Maybe` and `Either` did!

To preface these blog posts, I don't have a background in math, other than
the calculus and differential equations I did in an undergrad engineering
degree (which I don't think I've really used at all). I don't have a
background in abstract algebra nor category theory, so I will likely misuse
terminology or miss important rigorous points. I'm happy to be corrected, as
that helps me to solidify my understanding, much of which has been informally
self-guided. I'm trying to approach these topics from a "boots on the ground"
perspective - a layman's guide to what I know about functional programming,
and why I think it matters.

Functional programming is one of the most interesting and satisfying things
to learn in the world of software development. It has a long history, it's
backed by years of research by some extrememly smart people, and really the
sky is the limit on how much you can learn and grow. I find it much more
satisfying to learn and apply these powerful abstractions than to try to keep
up with
*JavaScript Framework Du Jour* :tm:. The key thing to remember is to not get
overwhelmed and to just take one step at a time, and build your knowledge up
over time.