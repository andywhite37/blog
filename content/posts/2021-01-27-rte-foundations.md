--- 
date: 2021-01-27T18:43:19-07:00
title: "TypeScript + fp-ts: ReaderTaskEither Foundations"
description: ""
categories: ["Software Development", "Functional Programming"]
tags: ["TypeScript", "fp-ts", "React"]
---

# `ReaderTaskEither<R, E, A>` Foundations

This post is meant to give some background information on the [ReaderTaskEither<R, E, A>](https://github.com/gcanti/fp-ts/blob/master/src/ReaderTaskEither.ts) type from [fp-ts](https://gcanti.github.io/fp-ts/).

## What is a `ReaderTaskEither<R, E, A>`?

To understand `ReaderTaskEither<R, E, A>` (aka `RTE`), it's important to understand some of the lower-level `fp-ts` "effect types" upon which `RTE` is built. Note that in fp-ts, some of these types might be encoded slightly differently than below, but the concepts should be the same. Also note that nearly all of the types and concepts below have a history that predates `fp-ts` (and TypeScript for that matter) by decades - many of these ideas come from languages like Haskell, Scala, PureScript, OCaml, and others.

## `IO<A>`

```typescript
type IO<A> = () => A
```

- Sync/async: **sync**
- Can fail: **no**
- Can depend on explicitly-declared contextual info: **no**

`IO<A>` represents a lazy, synchronous computation that, when run, may perform side effects and then produce a value of type `A`. The side effects might be involved with the production of the `A` value (e.g. reading from a `DOM` element, `localStorage`, etc.), or might be unrelated, "unobservable" side effect(s), like synchronously writing to a file, sending a message on a network connection, writing to the `DOM`, etc. `IO<A>` is intended to be used for synchronous effectful computations that are not expected to fail, as there is no way to represent failure other than `throw`ing an `Error` from the function, which is undesirable, and would be unexpected by the caller. If you have a synchronous, effectful operation that can fail, see `IOEither<E, A>`.

### Aside: `IO<void>`

`IO<void>` is a type which represents a side-effecting computation that produces no value at all - its sole purpose is to perform side effects. This could be something like writing to `localStorage`, writing a log message, dispatching a redux action, drawning to the screen, writing to the `DOM`, etc.

```typescript
const mySideEffect: IO<void> = () => {
  writeToDatabase('my-key', 'my-value')
  writeToLocalStorage('key', 'something')
  dispatch(myAction())
  document.getElementById('my-id').innerHTML = 'hi'
  console.log('hi')
}
```

### Aside: `Lazy<A>`

```typescript
type Lazy<A> = () => A
```

`Lazy<A>` (aka a "thunk") represents a synchronous computation that produces a value of type `A` when the function is called. By convention, the `Lazy<A>` type does not typically imply the presence of side effects - the conventional semantics of a type like `Lazy<A>` are more about the deferral of a potentially expensive, but typically pure computation.

`Lazy<A>` and `IO<A>` have the same type, so what's the difference? There's not really a difference other than semantics and convention - `Lazy<A>` would typically be expected to be used for a lazy, but pure computation, whereas `IO<A>` is used for deferring the execution of side effects.

One other reason for the existence of both of these types is that in `fp-ts` v1, the `IO<A>` type was encoded slightly differently than it is in v2+, so there actually was a type-level difference between `Lazy<A>` and `IO<A>` at that time. In fp-ts v2+, the encoding of `IO<A>` was changed to simply `() => A` (or the `interface` encoding of that function type), so there is no longer a practical difference in the types.

## `IOEither<E, A>`

```typescript
type IOEither<E, A> = IO<Either<E, A>>
                    = () => Either<E, A>;
```

- Sync/async: **sync**
- Can fail: **yes**
- Can depend on contextual info: **no**

`IOEither<E, A>` represents a synchronous, lazy computation that can produce a value of type `A`, or fail with an error of type `E`. This is intended to be used for synchronous effectful code that has the possibility of failure. The `IO<_>` part implies the presence of side effects, because if there were no effects, it would probably be better to just use `Either<E, A>`, or potentially `Lazy<Either<E, A>>`. The `Either<E, A>` part of the type allows for the representation of errors in the effectful computation.

## `Promise<A>`

- Sync/async: **async**
- Can fail: **yes** (with non-generic/non-polymorphic `any` error)
- Can depend on explicit contextual info: **no**

`Promise<A>` Represents an eagerly-executed (i.e. non-lazy) async computation that can eventually succeed with a value of type `A`, or fail with an error of type `any`. The computation starts to execute as soon as the `Promise<A>` is constructed, so the type is therefore not lazy, and not referentially transparent, which makes `Promise<A>` a less appealing choice for pure functional programming. `Promise<A>` also has an implicit memoization of the success or failure result.

Because the computation is async, there is no way to synchronously extract a value of type `A` directly out of a `Promise<A>`. In other words, there is no function of type `Promise<A> => A`. The value that is eventually produced by a `Promise<A>` can be accessed by chaining on a continuation callback via `.then((a: A) => ...)`. Or you can use the `await` approach, which is essentially just syntax sugar for `.then`.

## `Task<A>`

```typescript
type Task<A> = () => Promise<A>
```

- Sync/async: **async**
- Can fail: **no** (by convention)
- Can depend on contextual info: **no**

`Task<A>` represents a lazily-executed async computation that can eventually produce a value of type `A`. In fp-ts, `Task<A>` is currently implemented as a `Lazy<Promise<A>>` or `() => Promise<A>`, so technically, a `Task<A>` can fail "under the hood," but by convention, `Task<A>` is intended to be used for async computations that are not expected to fail. In other words, if you use a `Task<A>` for a computation, and the underlying `Promise<A>` fails, you've made a programming error in your choice of `Task<A>` as your effect type, and you've not likely made any attempt to handle errors from the `Task<A>`, so your program will likely and rightfully crash. The canonical example of an async operation that is not expected to fail is a basic deferred function call, like a `setTimeout`. If you are dealing with an async operation that has the possibility of failure, you should use `TaskEither<E, A>` instead.

So if `Task<A>` is just a lazy `Promise<A>`, why not just use `Promise<A>`? The reason is that a value of type `Task<A>` is a pure and referentially-transparent **description** of an effectful computation, whereas a value of type `Promise<A>` is an impure, referentially-opaque, already-running (or possibly already-completed) effectful computation. Another intuition is that a `Task<A>` is a "canned" or "freeze-dried" side effect that you can pass around, compose, substitute, etc., and then open or thaw it out it at the right time; whereas, a `Promise<A>` is the contents of the can - it's a little messier to pass around. The simple act of making the evaluation of the `Promise<A>` lazy makes `Task<A>` pure. When run, the `Task<A>` will perform impure side effects, but the `Task` itself is pure. The same idea applies to `IO<A> = () => A` compared to an effectful expression that produces an `A` - the act of deferring the side effects make the `IO<A>` type pure. This idea may take some time and hands-on practice to sink in. Purity and referential transparency are important concepts in pure functional programming because they allow you to make assumptions about the behavior of your program based on provable mathematical laws, perform substitutions of expressions, variables, and values both in actuality and mentally, and generally reason about the behavior of your program just by looking at the types. Without some of these principles, some of which are enforced by the compiler and some of which are followed by convention and discipline, you can't really make any assumptions about the behavior of a program, because any piece of code can do just about anything it wants at any time. By constraining ourselves to a set of well-behaved types and principles, we can eliminate whole classes of bugs and unexpected behaviors, and make our code more maintainable and reusable.

## `TaskEither<E, A>`

```typescript
type TaskEither<E, A> = Task<Either<E, A>>
                      = () => Promise<Either<E, A>>;
```

- Sync/async: **async**
- Can fail: **yes**
- Can depend on contextual info: **no**

`TaskEither<E, A>` represents a lazy async computation that can succeed with a value of type `A`, or fail with an error of type `E`. The underlying `Promise` can actually also fail "under the hood" with it's own unknown (`any`) error type, but by convention, errors in `TaskEither<E, A>` are meant to be lifted into the `Either<E, A>` in the `Promise`'s "success channel." If you end up with a failed Promise in a `TaskEither`, you've made a programming error by not lifting an error into the `Either<E, A>` somewhere, and your program will likely crash. This pitfall is an unfortunate consequence of using `Promise<A>` as the basis of the `Task`-based effects in fp-ts. However, `Promise<A>` is so ubiquitous in JavaScript and TypeScript, that by using `Promise<A>` under the hood, you gain the ability to more easily interop with most existing JavaScript libraries, and you can leverage some existing familiarity with `Promise<A>` for learning purposes. There are async effect libraries that are not backed by `Promise<A>` that can be explored for a better understanding of this.

Like `Promise<A>`, you can't "get the value out" of a `Task<A>` nor anything based on `Task`. You access the value by composing on functions like `map`, `chain`, and others.

## `Reader<R, A>`

```typescript
type Reader<R, A> = (r: R) => A
```

- Sync/async: **sync**
- Can fail: **no**
- Can depend on contextual info: **yes**

`Reader<R, A>` represents a synchronous computation that produces a value of type `A` by reading from some contextual value provided via the function argument of type `R`. A `Reader<R, A>` is just as simple as it looks - it's just a function that takes an argument `R` and produces a value `A`. The key aspect of `Reader` is encoding the ability for a computation to utilize an **explicit** input value to perform its computation. If you think about the effect types we've seen so far (`IO<A>`, `TaskEither<E, A>`, etc.), they only deal with outputs - the `E` error type or the `A` success type. `Reader` introduces the concept of an input, and gives you the power to compose effects that depend on different inputs to run. How `Reader` is used in practice is where it gets more interesting. Note that a `Reader<R, A>` does not typically imply the presence of side effects by itself - for something like that, you'd probably use `type ReaderIO<R, A> = Reader<R, IO<A>>`, `ReaderIOEither<R, E, A> = Reader<R, IOEither<E, A>>`, or the `ReaderTask*` types.

## `ReaderIO<R, A>` and `ReaderIOEither<R, E, A>`

```typescript
type ReaderIO<R, A> = Reader<R, IO<A>>
                    = Reader<R, () => A>
                    = (r: R) => () => A;

type ReaderIOEither<R, E, A> = Reader<R, IOEither<E, A>>
                             = Reader<R, () => Either<E, A>>
                             = (r: R) => () => Either<E, A>;
```

As mentioned above, these types simply combine the capabilities of `Reader<R, A>` with `IO<A>` or `IOEither<E, A>` - i.e. reading from some input value in order to perform a side-effectful computation.

Note that `ReaderIOEither` may not exist in `fp-ts` at the time of this writing, but it's easy to create by just following or copying how something like `ReaderTask<R, A>` or `ReaderTaskEither<R, E, A>` are implemented. See also [fp-ts-contrib](https://github.com/gcanti/fp-ts-contrib) for other variations like `StateTaskEither<S, E, A>`.

This approach of "stacking" effect types is similar to how monad transformers work, and is sometimes referred to as vertical composition of effects - a "stack of effects" or an "effect stack." The idea is that you create a "more capable" effect type (i.e. one that is able to handle more flexible or expressive effects) by combining the capabilities of less-capable effects.

## `ReaderTask<R, A>`

```typescript
type ReaderTask<R, A> = Reader<R, Task<A>>
                      = Reader<R, () => Promise<A>>
                      = (r: R) => () => Promise<A>
```

- Sync/async: **async**
- Can fail: **no**
- Can depend on contextual info: **yes**

As you might imagine, a `ReaderTask<R, A>` combines the capabilities of `Reader<R, A>` and `Task<A>`. `Reader` provides the ability to depend on some input to run, and `Task` provides the ability to perform an async computation that can't fail. To add the ability to fail, continue on to `ReaderTaskEither<R, E, A>`.

## `ReaderTaskEither<R, E, A>`

Below is a step-by-step expansion of `ReaderTaskEither<R, E, A>` into it's underlying type:

```typescript
type ReaderTaskEither<R, E, A> = Reader<R, TaskEither<E, A>>
                               = (r: R) => TaskEither<E, A>
                               = (r: R) => Task<Either<E, A>>
                               = (r: R) => () => Promise<Either<E, A>>
```

- Sync/async: **async**
- Can fail: **yes**
- Can depend on contextual info: **yes**

A `ReaderTaskEither<R, E, A>` is a type that combines the powers of a few of the less-capable effect types, into a type that provides the most commonly-needed capabilities for day-to-day application programming:

- `Reader` - grants the ability to depend on contextual information to perform a computation, _without having to have the information ahead of time._
  - Separates the description of the computation based on potentially abstract dependencies from the act of providing its concrete dependencies
  - `Reader` is the FP version of dependency-injection - you write your code using abstract dependencies that you expect to be given to you by some external caller or layer of your program. The dependencies often flow to the top-level of the program, where they are provided to all `Reader`-based effects right before running the whole effect stack to perform the computation.
- `Task` - grants the ability to perform async, side-effectful work
  - Separates the description of the async computation from the execution of it
- `Either` - grants the ability to perform a computation that can fail with a known error type

One high-level thing to note is that you can "provide" the `Reader` environment by passing in the `R` value, and you are left with a still-pure `TaskEither<E, A>` value, which you can continue to use in a pure way. The computation only runs when the `TaskEither<E, A>` is "run" (i.e. called), which finally constructs the `Promise<Either<E, A>>` and starts the effectful computation. You typically "run" an effect at the last possible moment, so that you can build your code using completely pure functions, and only drop down into impure execution at the very "end," like the end of a main program or the end of some context within your application, like when you pass control back to a framework (i.e. the end of a `DOM` event handler function), or you are leaving some context and the effect needs to happen at that time. The definition of this elusive "end" concept takes some time and hands-on practice to fully grasp.

## "Effect rotation"

One interesting property of `RTE` is the ability for it to represent some of it's less-capable underlying effect types, through creative application of its type params.

E.g. to denote an effect that has no `Reader` environment (e.g. an effect that doesn't depend on any contextual info), you can use the `unknown` type (or some other "empty" type) as the `R`, like:

```typescript
type NoEnvRTE<E, A> = ReaderTaskEither<unknown, E, A>
```

The `unknown` means that you can "provide" the environment required by this function by passing literally any value - it doesn't matter what it is, and nothing uses it. The `unknown` argument is simply there as a placeholder to satisfy the type. You might notice this looks like just a `TaskEither<E, A>`, and it essentially is - it is isomorphic with `TaskEither<E, A>`, which means you can convert this type to and from `TaskEither<E, A>` without losing anything.

To denote an async computation that can't fail, you can use `never` for the `E` type:

```typescript
type NoErrorRTE<R, A> = ReaderTaskEither<R, never, A>
```

For the error type, we use `never`, because the `never` type has no inhabitants, so it's impossible to ever get this type into the failed state, because there's no way to create a value of type `never`. This type is isomorphic with `Reader<R, Task<A>>` or `ReaderTask<R, A>`. This type is interesting because it can be used as a signal that you've "handled" any errors, probably by converting any possible errors in a value of type `A` in the success channel, and eliminating the need for any successive functions to deal with any errors at all.

For a computation that has no `Reader` env and that additionally can't fail, you can use `ReaderTaskEither<unknown, never, A>`, which is isomorphic with `Task<A>`. To reiterate, the reason we use `unknown` for the reader type and `never` for the `E` type is that we want to be able to provide this reader with any value to satisfy the reader argument, but we want to ensure that the effect can't fail by disallowing any value from appearing in the `Left` of the `Either<E, A>`. Ensuring an effect can't fail means that you have to handle any possible errors by expressing them as a value of type `A` in the success channel.

This ability for this one type `RTE<R, E, A>` to represent these different variations of effects was dubbed "effect rotation" by John De Goes, the creator of ZIO, [in one of his earlier articles](https://degoes.net/articles/rotating-effects) about the ideas behind `ZIO`. This is considered "rotation" because with monad transformers, you might create these effect combinations by (vertically) "stacking" different effect types, but with `RTE`, you achieve these effect capability variations by applying different types at a single, "flat" level (i.e. "horizontally"). (`RTE` itself is a stack of effect effect types, so the analogy isn't perfect, but the way the `RTE` type is expressed is is similar to `ZIO` in spirit).

## Composing effects

For all of the above effect types (`IO`, `Task`, `RTE`, etc.), it's possible to create `Functor`, `Applicative`, `Monad`, and a variety of other typeclass instances. These typeclass instances allow us to compose or combine our effectful computations into more complex computations and to eventually build whole programs based on `RTE`. If you've never done it before, it's useful to go through the exercise of implementing the following functions for all of the above types, e.g.:

```typescript
// Functor
const ioMap = <A, B>(f: (a: A) => B) => (io: IO<A>): IO<B> => { ... }

// Applicative
const ioOf = <A>(a: A): IO<A> => { ... }
const ioApply = <A, B>(ff: IO<(a: A) => B>) => (io: IO<A>): IO<B> => { ... }

// Monad
const ioChain = <A, B>(f: (a: A) => IO<B>) => (io: IO<A>): IO<B> => { ... }
```

Try implementing these for `IO<A>`, then try again with all the other effect types. This is an interesting exercise to see how function-based types work with operations like `map` and `chain`, compared to how the simpler data structures like `Option`, `Either`, and `RemoteData` work. The main guidance is to "follow the types" - think about what types you are given and what type of value you are trying to produce in the end.

# Conclusion

`ReaderTaskEither<R, E, A>` combines the capabilities of `Reader<R, A>`, `Task<A>` and `Either<E, A>` to create a type that can handle most effectful computations needed for application development. In another document, we'll explore more real-world usage examples and patterns of `RTE`.
