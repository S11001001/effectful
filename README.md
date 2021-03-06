# Effectful [![Build Status](https://travis-ci.org/pelotom/effectful.png?branch=master)](https://travis-ci.org/pelotom/effectful)
> If we want to rationalize the notational impact of all these structures, perhaps we should try to recycle the notation we already possess.

\- McBride, Paterson, [_Applicative programming with effects_](http://www.soi.city.ac.uk/~ross/papers/Applicative.html)

---

Effectful is a small macro library that allows you to write monadic code in a more natural style than that afforded by `for`-comprehensions, embedding effectful expressions in other effectful expressions rather than explicitly naming intermediate results. The idea is similar to that of the [Scala Async](https://github.com/scala/async) library, but generalized to arbitrary monads (not just `Future`).

## Introduction

The Effectful library provides two basic dual operations: `effectfully: A => M[A]` and `unwrap: M[A] => A` (there is also `!`, a postfix version of `unwrap`). Intuitively, within an `effectfully` block, we are allowed to treat impure (or *effectful*) values as if they were pure. If you think about it, this is exactly what it's like to program in a standard imperative programming language. For example, take this hypothetical code:
```scala
if (!db.lookup(key).isDefined)
  db.add(key, value);
```
where `db.lookup` and `db.add` do the obvious side-effectful things of interacting with a remote database. In order to reify the side-effects of this snippet in the type system, we can define a monad for our database type (or just use [`IO`](https://github.com/scalaz/scalaz/blob/scalaz-seven/effect/src/main/scala/scalaz/effect/IO.scala)). Then, in Scala we could write something like this instead:
```scala
for {
  optVal <- db.lookup(key)
  _ <- optVal map (db.add(key, _)) getOrElse db.pure(())
} yield ()
```
But this seems to have lost something of the perspicuity of the original. Effectful lets us write it in the original style but with all effects documented in the type system:
```scala
effectfully {
  if (!db.lookup(key).!.isDefined)
    db.add(key, value).!
}
```
Notice the use of the postfix `!` operator to indicate where effects are happening.

## Quick start

In your `build.sbt`, add a dependency like so:

    libraryDependencies += "org.pelotom" %% "effectful" % "1.0.1"

Now write some code using Effectful:

```scala
import scalaz._
import Scalaz._
import effectful._
import language.postfixOps

val xs = List(1,2,3)
val ys = List(true,false)

effectfully { (xs!, ys!) }

// ==> List((1,true), (1,false), (2,true), (2,false), (3,true), (3,false))
```

Here, the "effect" in question is nondeterminism.

## Nested effects

In Scala we have `for`-comprehensions as an imperative-looking syntax for writing monadic code, e.g.

```scala
for {
  x <- foo
  y <- bar(x)
  z <- baz
} yield (y, z)
```

Each monadic assignment `a <- ma` _unwraps_ a pure value `a: A` from a monadic value `ma: M[A]` so that it can be used later in the computation. But this is a little less convenient than one might hope--frequently we would like to make use of an unwrapped value without having to explicitly name it. With Effectful we can write it _inline_, like so:

```scala
effectfully { (unwrap(bar(unwrap(foo))), unwrap(baz)) }
```

or using `!`, simply

```scala
effectfully { (bar(foo!)!, baz!) }
```

### Effects within conditionals

Writing conditional expressions in `for` comprehensions can get hairy fast:

```scala
for {
  x <- foo
  result <- if (x > 12) {
    for {
      a1 <- bar
      a2 <- baz(a1)
    } yield a2
  } else {
    for {
      b1 <- boz
      b2 <- biz
    } yield b1 * b2
  }
} yield result
```

With Effectful we can write this as:

```scala
effectfully {
  if (foo.! > 12)
    baz(bar!)!
  else 
    (boz.! * biz.!)
}
```

Monadic `match`/`case` expressions are similarly easier to express with Effectful.

### Effects within `for`-loops and -comprehensions

We can even `unwrap` monadic values within loops; here's an example using the [`State` monad](http://www.haskell.org/haskellwiki/State_Monad):

```scala
effectfully {
  for (i <- xs; j <- ys)
    put(get[Int].! + 2 * i).!
}.run(n)
```

Compare with a traditional imperative loop that does the same thing, but with side-effects:

```scala
var v = n
for (i <- xs; j <- ys)
  v = v + 2 * i
```

Similarly, a `for`-comprehension containing `unwrap`s will sequence the effects of each monadic action it encounters, yielding all the results:

```scala
def fib(n: Int) = effectfully {
  for (i <- 1 to n) yield {
    val (x, y) = get[(Int, Int)].! // unfortunately we need to remind `get` what type of state it's dealing with
    put((y, x + y)).!
    x
  }
} eval (1, 1)

// fib(20) ==> List(1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987, 1597, 2584, 4181, 6765)
```

[Here's an alternate version](https://gist.github.com/pelotom/5474817) of the same function using Effectful with the [`ST` monad](http://www.haskell.org/haskellwiki/Monad/ST).

## How it works
    
Just as `for`-comprehensions transform your code into `flatMap`s and `map`s behind the scenes, `effectfully` is a macro which transforms code using `unwrap` / `!` into calls to `bind` and `pure` from Scalaz's `Monad` type class. So Effectful only works with instances of `Monad` (at the moment; there is [a proposal](https://github.com/pelotom/effectful/issues/2) to allow using superclasses of `Monad` where only their limited functionality is needed).

The transformation of `for`-loops and -comprehensions requires that your "iterable" type be an instance of the Scalaz `Traverse` type class. Then, the idea is that "effectful" loops and comprehensions (those which contain `unwrap` invocations) are transformed in the following way:
 - `t map (x => ...)` becomes `unwrap(t traverse (x => effectfully { ... }))` 
 - `t flatMap (x => ...)` becomes `unwrap(t traverse (x => effectfully { ... }) map (_.join))`
 - `t foreach (x => ...)` becomes `unwrap(t traverse (x => effectfully { ... }) map (_ => ()))`
 - `t withFilter {x => ...}` becomes `unwrap(t filterM (x => effectfully { ... }))`
 
The `flatMap` case implicitly adds the additional requirement that the "iterable" type have a `Monad` instance, which it ought to since you're `flatMap`ping it! And the `withFilter` case just requires that you be traversing a type which has a `filterM` method with the appropriate type; Scalaz defines this for `List` and, if you `import scalaz.std.indexedSeq.indexedSeqSyntax._`, any subtype of `scala.collection.immutable.IndexedSeq`.

## Limitations

Within the lexical scope of a `effectfully` block, not all invocations of `unwrap` / `!` are valid; in particular:
 - Function bodies cannot contain `unwrap` calls except in certain limited cases (anonymous functions passed to `map`, `flatMap`, `foreach` and `withFilter`).
 - By-name arguments cannot contain `unwrap` calls; these are essentially the same as function bodies.
 - It makes no sense to use `unwrap` outside of an `effectfully` block.

When `unwrap` is used in an unsupported position, it will be flagged with an error.
