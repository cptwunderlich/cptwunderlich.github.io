---
layout: post
title: Generating Tests using Scala 3 Macros
tags:
  - Scala
  - Scala3
  - programming
---

# Porting Old Code...

At work, I just finished getting our in-house libraries to cross-compile to Scala 2.13 and 3.3.
Of course, at some point I encountered a macro, and since the macro system got a complete overhaul
in Scala 3, I needed to rewrite that.

I've had almost no contact with Scala 2 or 3 macros before, so I wasn't quite looking forward to the task.
Co-Pilot produced some good looking code, which had only one compilation error. However, after trying to get it to
work for a few hours, I realized, that the code was rubbish.
(See, AI won't come for our jobs just yet ;) )

# The Task

## Goal

This particular macro is intended to generate test cases for translation keys.
You define an object with a bunch of translation keys of type `TranslationKeyN[M, I1..IN]`,
where `N` is the arity of the key (how many interpolation parameters it has), `M` is a marker trait and
`I1..IN` the types of the interpolation values.
This is to make sure, that each key has a translation with the appropriate number of parameters.

## Steps

Alright, so we have a test class with an overloaded method, one for each arity.
Our users call the macro with some container object `C`, which contains the keys for some
marker trait (i.e., the keys that belong grouped together, e.g., for some email).

We need to:

  - Enumerate the members of `C`
  - Filter the public value definitions (`val`s)
  - Filter the ones of the right type (`TranslationKeyN`)
  - Generate an expression calling the right overload

The details of the implementation look a bit complex, but I found it an interesting example for a macro,
because we deal with reflection (enumerating members), calling methods on instances, overloaded methods even,
type arguments and even contextual abstractions (implicits/using/given).

# A First Try

This article is not going to be an in-depth introduction to Scala 3 macros.
But here are some helpful resources:

  - [The official Tutorial](https://docs.scala-lang.org/scala3/guides/macros/index.html)
  - [Scala 3 macros tips & tricks (Softwaremill)](https://softwaremill.com/scala-3-macros-tips-and-tricks/)
  - [lampepfl: dotty-macro-examples](https://github.com/lampepfl/dotty-macro-examples)
  - [Actual Scala 3 library source code](https://github.com/scala/scala3/blob/3.5.0/library/src/scala/quoted/Quotes.scala) (this is, probably, the most useful one!)

Alright, so here is an outline of our `TranslationTest` class:

```scala
class TranslationsTest[A]:
  def testTranslationKey(key: TranslationKey0[A]): Unit =
    testKeyInternal(key.value)(key()(_))

  def testTranslationKey[I1: Arbitrary](key: TranslationKey1[A, I1]): Unit =
    testKeyInternal(key.value)(key(random[I1])(_))

  // Provide more overloads...

  inline def testAllTranslationKeys[C](container: C): Unit =
    ${ macroImpl[A, C]('this, 'container) }
```

The method `testKeyInternal` will use the translation bundles to generate a table driven test.
But in my example, it just does some `println`-ing.
Using the `random` function, we generate some random input for the interpolation arguments,
i.e., for a key "`Welcome {{user.name}}`" of type `TranslationKey1[User, String]`,
we need an `Arbitrary[String]` instance to generate some random value.
This comes from [scalacheck](https://github.com/typelevel/scalacheck/blob/main/doc/UserGuide.md).

In the original code, `TranslationsTest` serves as a base-class for tests and a call to the
`inline` method will generate the test cases there.

The method is the entry point to the macro and contains a splice with the call to the macro.
We are passing quoted references to `this` and the container object with the keys.
(extra credit: you don't need to pass `this`, because you can get a reference to the owner of a splice.
In my case, it wasn't necessary, because this is library internal code and the user facing code would be the same.)

Let's give the actual macro a try now. The implementation has to be in a different compilation unit -
in my example I put it into `Macro.scala`:

```scala
import scala.quoted.*

def macroImpl[T, C](
    testsExpr: Expr[TranslationsTest[T]],
    container: Expr[C]
)(using quotes: Quotes, tTpe: Type[T], cTpe: Type[C]): Expr[Unit] =
  import quotes.reflect.*

  val tkTypes = List(
    TypeRepr.of[TranslationKey0[T]],
    TypeRepr.of[TranslationKey1[T, _]],
    TypeRepr.of[TranslationKey2[T, _, _]],
    TypeRepr.of[TranslationKey3[T, _, _, _]],
    TypeRepr.of[TranslationKey4[T, _, _, _, _]]
  )

  def isTranslationsKey(t: TypeRepr): Boolean =
    tkTypes.exists(t <:< _)

  val containerType: TypeRepr = TypeRepr.of[C]
  val containerTerm: Term = container.asTerm
```

Let's have a look at the start of the function.
We have our type arguments for the maker type `T` and the container type `C`.
The first parameter is our expression containing `this` and the second one our container object.
We need to use `Quotes` because this is a macro and we need instances of `Type[A]` for any type `A`
we want to reference in our quote.
The way I understand this, this is due to the JVM using type erasure for generics and if we want
to actually do something with the type, it needs to be reified.

Alright, so the rest is mostly machinery to check that a member value has the type we are looking for.
Then we get a type representation of `C`, which we can use for reflection and a term of the container,
so we can use that to access the actual members of it.

Moving on:

```scala
val calls = containerType.typeSymbol.fieldMembers.map { symbol =>
  // Val definition is `val name: tpt = _rhs`
  case ValDef(name, tpt, _) if isTranslationsKey(tpt.tpe) => {
    val field: Term = Select(containerTerm, symbol) // container.`name`
    Some('{$testsExpr.testTranslationKey($i{field.asExpr})})

  case _ => None
}.flatten
```

We just get the value definitions that match and access the value on the actual container,
passing it to the function we want to call in quoted code. Easy.

# The Actual Solution

Unfortunately, this doesn't work...

I was hoping it would just figure out the correct overload, but the expression we get from the term `field`
is just an `Expr[Any]`.
I tried a few things to get this to work, but then I gave up and just constructed the AST manually,
applying the type arguments, arguments and resolving the givens (implicits):

```scala
  // Map over all member fields of container type,
  // searching for all translation keys.
  val calls = containerType.typeSymbol.fieldMembers.map { symbol =>
    symbol.tree match
      // Val definition is `val name: tpt = _`
      case ValDef(name, tpt, _) if isTranslationsKey(tpt.tpe) => {
        val field: Term = Select(containerTerm, symbol) // container.`name`
        val interpolationTArgs = tpt.tpe.typeArgs.tail // I1, I2, I3, ...

        // Resolve the needed `testTranslationKey` overload
        val callOverload = Select.overloaded(
          testsExpr.asTerm,
          "testTranslationKey",
          interpolationTArgs,
          List(field)
        )

        // Resolve givens/implicit for Arbitrary value generation
        val arbitraryTpe =
          TypeIdent(Symbol.requiredClass("org.scalacheck.Arbitrary")).tpe
        val implicits = interpolationTArgs.map { t =>
          Implicits.search(arbitraryTpe.appliedTo(t)) match
            case success: ImplicitSearchSuccess =>
              success.tree
            case failure =>
              report.errorAndAbort(s"Could not find implicit: $failure")
        }
        val res =
          if (implicits.nonEmpty) Apply(callOverload, implicits)
          else callOverload

        Some(res.asExpr)
      }
      case _ => None
  }.flatten
```

We need the final conditional, because at arity 0 (`TranslationKey0`) we don't have any implicits to apply.

Finally, we return a code block with all the calls:

```scala
// Block expression with all calls
Expr.block(calls, '{ () })
```

You can find the whole code in [this GitHub repository](https://github.com/cptwunderlich/translation-keys-macro).

# Conclusion

Scala 3 macros are very powerful, but it takes some time getting used to thinking in this "meta" way.
Furthermore, there aren't a ton of resources online about this topic. Maybe this article can help someone trying
to solve a similar problem.

If you have any suggestions on how to improve this macro, or generals suggestions, as well as questions,
just contact me via Twitter or [the GitHub discussion for this post](https://github.com/cptwunderlich/cptwunderlich.github.io/discussions/2).
