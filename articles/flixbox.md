---
title: Barebones web app example using fp-ts
description: Barebones web app example using functional programming library fp ts in TypeScript
cover_title: flixbox
tags: typescript,fp
published: 2023-01-25T12:41:00
updated: 2023-05-29T12:42:00
---

[![flixbox - Search movie trailers](./flixbox.jpg)](https://ogu.nz/wr/flixbox.html)

> [**flixbox**](https://www.github.com/onur1/flixbox) demonstrates a full stack client/server web application for interacting with the [TheMovieDB](https://www.themoviedb.org/) API using typed functional programming library [fp-ts](https://gcanti.github.io/fp-ts/) and its friends.

[fp-ts](https://gcanti.github.io/fp-ts/) is a library by [Giulio Canti](https://twitter.com/giuliocanti) that brings the power of [typeclasses](https://en.wikipedia.org/wiki/Type_class) and the [higher kinded types](https://en.wikipedia.org/wiki/Kind_(type_theory)) from functional programming languages (such as [Haskell](https://www.haskell.org/) and [PureScript](https://www.purescript.org/)) into the world of [TypeScript](https://www.typescriptlang.org/).

All of the functionality in the application above, including the server side API, is implemented using libraries from the [fp-ts module ecosystem](https://gcanti.github.io/fp-ts/ecosystem/). These are all very cool ideas from all around the FP world. In this post, we will walk through this example and see what makes these functional modules so great.

## Overview of the HTTP API

#### Requests and data formats

All requests to the flixbox API are HTTP GET requests. API responses are only available in JSON format. No authentication required.

#### Errors

When something goes wrong, flixbox will respond with the appropriate HTTP status code and an error. This can be one of:

- Validation error: User input couldn't be validated.
- Provider error: TMDb failed to respond with valid payload.
- Not found: Requested resource not found.
- Server error: Generic server error.
- Method error: Method not allowed.

## Searching movies

```
GET /results?search_query=QUERY
```

Responds with a [`SearchResultSet`](https://github.com/onur1/flixbox/tree/0.0.7/src/tmdb/model/SearchResultSet.ts) object.

## Retrieving a movie

```
GET /movie/ID
```

Responds with a [`Movie`](https://github.com/onur1/flixbox/tree/0.0.7/src/tmdb/model/Movie.ts) object.

## Get popular movies

```
GET /popular
```

Responds with a [`SearchResultSet`](https://github.com/onur1/flixbox/tree/0.0.7/src/tmdb/model/SearchResultSet.ts) object.

# HTTP middleware architecture

> The [server](https://github.com/onur1/flixbox/tree/0.0.7/src/server) API is implemented using [hyper-ts](https://github.com/DenisFrezzato/hyper-ts): the fp-ts porting of [Hyper](https://hyper.wickstrom.tech/). This is an experimental middleware architecture which enforces _strict ordering_ of middleware compositions using static type-checking.

Under the hood, hyper-ts runs [Express](https://expressjs.com/) server, but you can integrate it with any HTTP server you like.

Hyper is modeled as a [State monad](https://paulgray.net/the-state-monad/) &mdash;you can think of it as the combination of [Reader](https://dev.to/gcanti/getting-started-with-fp-ts-reader-1ie5) and [Writer](https://levelup.gitconnected.com/reader-writer-and-state-monad-with-fp-ts-6d7149cc9b85) monads, the kind of [monads](https://dev.to/gcanti/getting-started-with-fp-ts-monad-6k) which allow you to read/write values from/to an environment in a monadic fashion. In this case, it reads information about the incoming request and writes a response through the Express API.

The main principle is that it doesn't immediately mutate the connection (by writing headers or etc.), but it outputs a list of actions to run in strictly correct order (otherwise your code wouldn't have compiled in the first place) when the middleware has finished processing a request. This concept is also really helpful [while testing](https://github.com/onur1/flixbox/blob/0.0.7/__tests__/server.ts) your applications.

In the example below, you can see the entire pipeline for handling requests to the `/movie/ID` endpoint, it proxies requests to TMDb with caching support.

When [`/movie/3423`](https://onurgunduz.com/flixbox/movie/3423) is called on the flixbox API:

* The server checks the internal cache first:
  * If this movie is already found there, it returns the cached value.
  * Otherwise it calls the TMDb API to retrieve it and saves the result into the cache, returning the newly cached value.
* Responds with a JSON object if the data retrieval succeeded in one way or another.

[`server/Flixbox.ts`](https://github.com/onur1/flixbox/blob/0.0.7/src/server/Flixbox.ts#L87)

```typescript
pipe(
  GET,
  // continue if this is a GET request only
  H.apSecond(
    pipe(
      // retrieve the requested entry from cache
      get(store, `/movies/${String(route.id)}`),
      H.map(entry => entry.value),
      // if not exists, fetch from TMDb
      H.orElse(() =>
        pipe(
          movie(tmdb, route.id),
          H.chain(value =>
            pipe(
              // insert the TMDb response to cache
              put(store, `/movies/${String(route.id)}`, value),
              H.map(entry => entry.value)
            )
          )
        )
      )
    )
  ),
  // write JSON response
  H.ichain(res =>
    pipe(
      H.status<AppError>(200),
      H.ichain(() => sendJSON(res))
    )
  )
)
```

Here, the function we passed into `apSecond` only executes if the preceding `GET` middleware succeeds, and the function we passed into `orElse` only executes if the preceding `get` call fails.

The main pipeline will short-circuit with an [`AppError`](https://github.com/onur1/flixbox/blob/0.0.7/src/server/Error.ts) if any of the inner pipelines fails for some reason, and exit without writing a response.

Let's see the definition of `GET` (essentially `method`) middleware which is the initial middleware used in the example above.

[`middleware/Method.ts`](https://github.com/onur1/flixbox/blob/0.0.7/src/server/middleware/Method.ts)

```typescript
import { right, left } from 'fp-ts/lib/Either'
import { StatusOpen } from 'hyper-ts'
import { decodeMethod, Middleware } from 'hyper-ts/lib/Middleware'
import { MethodError, AppError } from '../Error'

function method<T>(name: string): Middleware<StatusOpen, StatusOpen, AppError, T> {
  const lowercaseName = name.toLowerCase()
  const uppercaseName = name.toUpperCase()
  return decodeMethod(s =>
    s.toLowerCase() === lowercaseName
      ? right<AppError, T>(uppercaseName as T)
      : left(MethodError)
  )
}

export const GET = method<'GET'>('GET')
```

The `method` middleware compares the incoming request method with the provided method name in the lowercase form and outputs it in the uppercase form if they match, otherwise it throws a `MethodError` (which is a kind of `AppError`). This middleware can only be composed with other middlewares if the initial connection state `StatusOpen` has not changed yet, which means you can only compose this with other middlewares if you haven't written a header or response yet.

Like `method`, all middlewares in the main pipeline return an `AppError`:

* `get` returns a `NotFoundError` when an entry is not found.
* `put` returns a `ServerError` when an entry couldn't be saved.
* `movie` fails with `ProviderError` that encapsulates the TMDb API errors.

If I pipe the result of this middleware pipeline into another `orElse` call and compose it with an error handler middleware as the final thing, then I can handle the `AppError` it throws very conveniently, and eventually send the appropriate error message (with sensitive information redacted) or log important errors. The [`destroy`](https://github.com/onur1/flixbox/blob/0.0.7/src/server/middleware/Error.ts) middleware just does that.

# Logging

> While we're at it, the logging functionality is based on the [logging-ts](https://github.com/gcanti/logging-ts/) module which is adapted from [purescripting-logging](https://github.com/rightfold/purescript-logging).

This is a very light-weight logging solution for creating composable loggers. I [wired it up](https://github.com/onur1/flixbox/blob/0.0.7/src/logging/TaskEither.ts) with hyper-ts over a [`TaskEither`](https://gcanti.github.io/fp-ts/modules/TaskEither.ts.html) instance, but I don't see any reason why the Middleware itself couldn't be used to implement the [`Console`](https://github.com/onur1/flixbox/blob/0.0.7/src/logging/Console.ts).

# Runtime type system

> If I had to choose only one thing from the fp-ts toolstack, that would be [io-ts](https://github.com/gcanti/io-ts/).

Both the server and the client use this library extensively for type validation.

To name a few use cases,

- The [client application state](https://github.com/onur1/flixbox/blob/0.0.7/src/app/Model.ts) is defined with it.
- [TMDb data model](https://github.com/onur1/flixbox/tree/0.0.7/src/tmdb/model) is defined with it, too.
- It is used for [reporting validation errors](https://github.com/onur1/flixbox/blob/0.0.7/src/server/Error.ts#L17).
- Routers use it for [matching queries](https://github.com/onur1/flixbox/blob/0.0.7/src/app/Router.ts#L5).
- React components use it with [prop-types-ts](https://github.com/gcanti/prop-types-ts/) for [validating received props](https://github.com/onur1/flixbox/blob/0.0.7/src/app/components/Layout.tsx#L77).
- [Environment variables](https://github.com/onur1/flixbox/blob/0.0.7/src/server/index.ts#L72) are validated with it.

There are many type validation libraries out there, but there must be a reason why the ones written by Giulio Canti (previously [tcomb](https://github.com/gcanti/tcomb) as well) became so popular and widely adopted in the JS community.

The reason is that other libraries are full of design mistakes which cause [type inference](https://en.wikipedia.org/wiki/Type_inference) to work poorly. You can't just _invent_ a technique for composing types, you can only _discover_ such things; and that discovery was made decades ago, io-ts is simply implementing that.

# Optics &mdash;i.e. immutable state updates

> [monocle-ts](https://www.github.com/gcanti/monocle-ts) is a partial porting of [Monocle](https://www.optics.dev/Monocle/) from Scala. It is used in the client application for reading and transforming the application state.

This library provides support for [composable optics](https://medium.com/@gcanti/introduction-to-optics-lenses-and-prisms-3230e73bfcfe) that are used for reading and writing immutable data. Simply told, you can create such an optic structure (perhaps a [Lens](https://gcanti.github.io/monocle-ts/modules/Lens.ts.html) composition) to zoom into a deeply nested object for transforming or reading a value inside it without touching the original value.

There are other libraries such as [Immer.js](https://immerjs.github.io/immer/) for doing this type of stuff. It gives you this `produce` function that you can use to change a value inside some object and return a copy.

```javascript
import produce from "immer"

// curried producer:
const toggleTodo = produce((draft, id) => {
    const todo = draft.find(todo => todo.id === id)
    todo.done = !todo.done
})

const nextState = toggleTodo(baseState, "Immer")
```

Optics do a similar thing, but in a type-safe composable fashion. This is one of the ways how you would program the same functionality in monocle-ts using a [`Traversal`](https://gcanti.github.io/monocle-ts/modules/Traversal.ts.html):

```typescript
import * as _ from 'monocle-ts/lib/Traversal'

type T = { id: number; done: boolean }

type S = ReadonlyArray<T>

const getNextState = (id: number) =>
  pipe(
    _.id<S>(),
    _.findFirst(n => n.id === id),
    _.prop('done'),
    _.modify(done => !done)
  )

const nextState = getNextState(42)(baseState)
```

# Routing

On this page, the URL in the address bar is synced with the flixbox window. You can actually [visit the current page with an initial route](./flixbox.html#/movie/545611).

> Both the client and the server use [fp-ts-routing](https://github.com/gcanti/fp-ts-routing) for parsing request routes. It is a cross-platform library and stacks with io-ts very nicely.

```typescript
import * as t from 'io-ts'
import { lit, query, int, zero } from 'fp-ts-routing'

// popular matches /popular.
const popular = lit('popular')

// movie matches /movie/ID.
const movie = lit('movie').then(int('id'))

// SearchQuery is an io-ts type for matching the query part of a URL.
const SearchQuery = t.interface({
  search_query: t.union([t.string, t.undefined]),
})

// results matches /results?search_query=WORD.
const results = lit('results').then(query(SearchQuery))
```

# Poor man's Elm in TypeScript

> [Elm](https://elm-lang.org/) is a programming language designed specifically for programming GUIs. [elm-ts](https://github.com/gcanti/elm-ts) is the fp-ts adaptation of it built on top of [RxJS](https://rxjs.dev/).

Note that elm-ts works like Elm only on the surface, otherwise internally they are totally different. Also, the Elm language uses the Hindley Milner type system [which is quite different](https://dev.to/lucamug/typescript-and-elm-3g38) from TypeScript's own type system.

There is an entire literature about [Functional Reactive Programming](https://en.wikipedia.org/wiki/Functional_reactive_programming) (FRP) and the [Elm paper](https://elm-lang.org/assets/papers/concurrent-frp.pdf) by [Evan Czaplicki](https://github.com/evancz) is a good start if you want to dig in deeper. For those interested, I would also recommend taking a look at [purescript-behaviors](https://github.com/paf31/purescript-behaviors) by [Phil Freeman](https://functorial.com/) which implements [push-pull FRP](http://conal.net/papers/push-pull-frp/) in PureScript and has been ported to fp-ts too by Giulio Canti, under the name [behaviors-ts](https://github.com/gcanti/behaviors-ts).

[Elm is very similar to Redux](https://redux.js.org/understanding/history-and-design/prior-art). The terms, Message and the Update function in Elm are analogous to Action and Reducer in Redux.

Basically, you provide an initial state to it, a pure _view_ function for drawing visual elements (based on the current state), and a pure _update_ function which becomes responsible for transforming the application state when something happens.

The Flixbox UI defines the following messages. These are the only side-effects that can occur while you are browsing the app.

[`src/app/Msg.ts`](https://github.com/onur1/flixbox/tree/0.0.7/src/app/Msg.ts)

```typescript
type Msg =
  | Navigate
  | PushUrl
  | UpdateSearchTerm
  | SubmitSearch
  | SetHttpError
  | SetNotification
  | SetSearchResults
  | SetPopularResults
  | SetMovie
```

When you dispatch one of these actions from your views (for example when a link is clicked and `Navigate` is triggered), the [update function](https://github.com/onur1/flixbox/blob/0.0.7/src/app/Effect.ts#L64) is called with a particular type of message and the current application state as input.

```typescript
function update<S, A>(msg: A, state: S): [S, A]
```

As you see, `update` takes a `msg` which has type `A` as its first parameter, and a `state` with type `S` as the second, returning both a new state and an action to run in the next loop.

You send the new state to [subscribers](https://package.elm-lang.org/packages/elm/core/latest/Platform-Sub) (such as the `view` function), and continue to process new actions until there is [nothing else to do](https://package.elm-lang.org/packages/elm/core/latest/Platform-Cmd). This pattern, as simple as it may seem, when compared to the traditional MVC, is actually a very powerful way to model state changes in UIs, to test and [debug](https://en.wikipedia.org/wiki/Time_travel_debugging) them.

# Conclusion

[PureScript and Haskell](https://gcanti.github.io/fp-ts/guides/purescript.html) are very elegant and concise programming languages. fp-ts is only emulating them and it has to deal with all the nitty gritty details to make this work with TypeScript types; while keeping the API up to date to not fall behind the developments within TypeScript, or the greater JavaScript ecosystem.

Working with fp-ts may feel like working in a construction zone sometimes, with coils of cables lying around everywhere and the loud [V8](https://v8.dev/) engine sound in the background; but once you get the hang of it, those cables or the noise doesn't bother you too much, because everything works flawlessly and nobody has to wear helmets in this worksite.
