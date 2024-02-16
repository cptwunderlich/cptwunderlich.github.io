---
layout: post
title: Automagic Scala Migrations
tags:
  - scala
  - scalafix
  - scalameta
  - scala-steward
  - programming
---

# Automagic Scala Migrations

## What & Why

We have an internal library and sbt plugin, which pins many library versions
and provides some base functionality.
The latest release features several larger updates, most notably Play 2.9.

However, there is a lot of churn related to library updates.
We use the [scala-steward Github action](https://github.com/scala-steward-org/scala-steward-action) to keep our services up-to-date,
but when there are code changes, incompatible versions, etc. a human has to intervene.

This new update in particular had a few mechanic changes related to Play 2.9 and a newer [Tapir version](https://github.com/softwaremill/tapir/releases/tag/v1.9.0).
So to make this update less painful for my colleagues, I decided to invest some time and figure out how to build [scalafix](https://scalacenter.github.io/scalafix/) migrations.

I was fantasizing about an automatic process, in which this update would get applied, merged and just magically work,
without anyone taking much notice. Alas, my expectations were soon dampened, but I did not expect it to go as badly as it did...
Anyway, I still believe that this is an awesome tool and the investment was worthwhile and will come in handy in the future.

This report is intended as a supplement to the tools' documentation and might help others in their journey to automatic scala migrations - and maybe a reader will have some suggestions for me.

## The Project Setup

### The Tools

For our scala-steward Github action to run migrations, we first have to write them. The tool used to create and apply linters and migrations is scalafix. It is based on [scalameta](https://scalameta.org/), a library to represent and query scala programs as syntax trees, as well as storing semantic information, like types and symbols, in a database format (SemanticDB).

### The Requirements

I did not set out to write some generally applicable scalafix rules, which should be published for anyone to use.
The migration should be very specific to our company's needs. Our artifacts are published to our own, private Sonatype Nexus Repository and our code is in our private GitHub Enterprise organization.

So I have to work with these constraints. That means, that all the tools need to be able to resolve artifacts from the private repo and GitHub resources cannot be accessed without authentication.

### Project structure

Scalafix allows you to use predefined rules (both internal and community release ones), but we don't care about this for now.

[The developer guide](https://scalacenter.github.io/scalafix/docs/developers/setup.html) tells us about our options to setup the project. You might want to have the rules as a sub-module in your project's sbt build, or as a separate project all together.

Rules can be consumed in different forms by scalafix. You can publish them as an artifact, like any other scala code. Or you can consume their source code and let scalafix compile them on the fly. Sources can be read via the `file:`, `http:` and `github:` protocols. The [tutorial](https://scalacenter.github.io/scalafix/docs/developers/tutorial.html#run-the-rule-from-source-code) contains some more information on this.

I chose to add them to the existing libraries project, as a sub-module. The migration is split up into several rules, each one taking care of a specific change. Rules are published to our Nexus repository as a compiled artifact.

Using the `github:` or `http:` schemes, I would have had to publish the rules on the public internet, e.g., in a gist on my personal GitHub account, because those do not work with authentication. That's a big no-no.

So the folder structure started out like in the tutorial. A `scalafix` folder in my project folder and inside the `input`, `output`, `rules` and `tests` folders.
The default package name for rules you find in the tutorial (and basically all rules you find linked) is `fix`.
As I thought this is just the first migration of many, and all of them will live in this project, I named my package `v2_5` - because it is the migration to version 2.5.
If you do that and you use the `github:` scheme to run rules from source, be aware that you have to [specify the package path](https://scalacenter.github.io/scalafix/docs/developers/tutorial.html#using-github).

Also make sure to use the correct package name in the `resources/META-INF/services/scalafix.v1.Rule` file.

The `input` directory holds your test files, which should be rewritten. After applying a rule from `rules`, the result should match the corresponding file in `output`. All of this is orchestrated by the test runner in `tests`.

### Cross Compilation

In our case, I wanted to rewrite the `Dependencies.scala` of our sbt build. One of the changes was renaming an artifact. Tapir upgraded from Play 2.8 straight to Play 3.0, which replaces Akka with Pekko, [in a bugfix release](https://github.com/softwaremill/tapir/releases/tag/v1.8.3)[^1]. So I needed to change that to the newly created [play29 modules](https://github.com/softwaremill/tapir/pull/3313) `tapir-play29-server`, etc.

To test this rule, the input file needs some sbt dependencies (e.g., for the `%%` functions). Sadly, sbt is still stuck on Scala 2.12.
So I needed to support Scala 2.12 and 2.13 inputs (I skipped Scala 3 for now).

I was struggling quite a bit with the correct setup, so here the short version for you.
I'm building the scalafix rules themselves with Scala 2.13 and the inputs and outputs are cross-compiled.
Dependencies are managed on a per-Scala-version basis. Here is a slightly edited version of the relevant sections from `build.sbt`:

```scala
val sbtScalaVersion       = "2.12.18"
val scala2_13             = "2.13.12"
val scalafixCrossVersions = sbtScalaVersion :: List(scala2_13)

lazy val `scalafix-input` = (project in file("scalafix/input"))
  .settings(
    crossScalaVersions := scalafixCrossVersions,
    evictionErrorLevel := Level.Warn,
    publish / skip := true,
    libraryDependencies ++= {
      CrossVersion.partialVersion(scalaVersion.value) match {
        case Some((2, 13)) =>
          Seq(
            "com.softwaremill.sttp.tapir" %% "tapir-play-server"     % "1.8.2",
            "com.typesafe.play"           %% "play-slick"            % "5.0.2",
            "com.lightbend.akka"          %% "akka-persistence-jdbc" % "4.0.0"
          )
        case Some((2, 12)) => Seq("org.scala-sbt" % "sbt" % "1.9.8")
        case _             => Seq.empty
      }
    }
  )
  .disablePlugins(ScalafixPlugin)

lazy val `scalafix-output` = (project in file("scalafix/output"))
  .settings(
    crossScalaVersions := scalafixCrossVersions,
    publish / skip := true,
    libraryDependencies ++= {
      CrossVersion.partialVersion(scalaVersion.value) match {
        case Some((2, 13)) =>
          Seq(
            "com.softwaremill.sttp.tapir" %% "tapir-play29-server"   % "1.9.8",
            "com.typesafe.play"           %% "play-slick"            % "5.2.0",
            "com.lightbend.akka"          %% "akka-persistence-jdbc" % "5.3.0"
          )
        case Some((2, 12)) => Seq("org.scala-sbt" % "sbt" % "1.9.8")
        case _             => Seq.empty
      }
    }
  )
  .disablePlugins(ScalafixPlugin)

lazy val `scalafix-rules` = (project in file("scalafix/rules"))
  .settings(commonSettings)
  .settings(
    name               := "underpin-scalafix-rules",
    scalaVersion       := scala2_13,
    crossScalaVersions := scalafixCrossVersions,
    scalacOptions += "-Ywarn-unused",
    libraryDependencies +=
      "ch.epfl.scala" %%
        "scalafix-core" %
        _root_.scalafix.sbt.BuildInfo.scalafixVersion
  )
  .disablePlugins(ScalafixPlugin)

lazy val `scalafix-tests` = (project in file("scalafix/tests"))
  .settings(
    crossScalaVersions := scalafixCrossVersions,
    publish / skip := true,
    evictionErrorLevel := Level.Warn,
    scalafixTestkitOutputSourceDirectories :=
      (`scalafix-output` / Compile / sourceDirectories).value,
    scalafixTestkitInputSourceDirectories :=
      (`scalafix-input` / Compile / sourceDirectories).value,
    scalafixTestkitInputClasspath :=
      (`scalafix-input` / Compile / fullClasspath).value,
    scalafixTestkitInputScalacOptions :=
      (`scalafix-input` / Compile / scalacOptions).value,
    scalafixTestkitInputScalaVersion :=
      (`scalafix-input` / Compile / scalaVersion).value
  )
  .dependsOn(`scalafix-input`, `scalafix-rules`)
  .enablePlugins(ScalafixTestkitPlugin)
```

This means that I have all the sources for the rules in the usual `src/main/scala` folder,
but the input and output sources are separated in `src/main/scala-2.12` and `src/main/scala-2.13` respectively.

## Writing Rules

This is the actual fun part!
Two important concepts to understand, is the [Syntax Tree](https://scalameta.org/docs/trees/guide.html#what-is-a-syntax-tree) and Tokens.
The former one is a tree representation of your program, which you will write matchers against.
Tokens are the building blocks of your program and represent keywords, identifiers and even indentation etc. We typically want to modify these.

The [scalafix guide ](https://scalacenter.github.io/scalafix/docs/developers/setup.html) describes how to write rules and there are many existing rules you can learn from.

### Syntactic vs. Semantic Rules

First, we need to decide whether we want to write a _syntactic_ or _semantic_ rule.

A _syntactic_ only needs a parsed program, i.e., the program structure and won't need a semanticDB with resolved types and symbols. I'd recommend to use syntactic rules whenever possible. They are faster and they work on programs with (semantic) compilation errors. For example, you update some dependencies and multiple changes are necessary to get the program to compile again. I might be biased, because I had huge problems with semantic rules in this regard. You might also get a `stale semanticDB` error with semantic rules, if the code was modified but not compiled/a new semanticDB generated in the meantime.

Syntactic rules are, however, less _powerful_. For example, if your code contains unqualified uses of two types with the same name, e.g., `com.someframework.Url` and `org.other.Url`, your rule might only see an identifier `Url`. It's up to you to make sure you are rewriting the correct ones. This might be easy, if you know your codebase and you can ascertain that such situations won't come up, simply by grepping through your code base.

_Semantic_ rules, on the other hand, have access to the resolved types and symbols. So you can, e.g., use scalafix' [SymbolMatcher](https://scalacenter.github.io/scalafix/docs/developers/symbol-matcher.html) to look for occurrences of `com.someframework.Url`.
Depending on how you run your scalafix rules, you'll need to [set your build to generate semanticDB files](https://scalacenter.github.io/scalafix/docs/users/installation.html#sbt), e.g.,

```scala
semanticdbEnabled := true, // enable SemanticDB
semanticdbVersion := scalafixSemanticdb.revision, // only required for Scala 2.x
```

### What to Match

I'm typically starting out by building a minimal example of what I want to rewrite (as input)
and what I would like it to look like afterwards (the output).
If you can think of any edge cases, or similar looking code you _don't want to rewrite_, put that into the test as well.

Then paste the code into the [Scalameta AST Explorer](https://scalameta.org/docs/trees/astexplorer.html), to get an idea what the syntax tree looks like.
Besides that, I found looking at the Scalameta [source for Tree nodes](https://github.com/scalameta/scalameta/blob/9df6bbfc4d9bd67ae6e0a51283dee3d5f944d7c5/scalameta/trees/shared/src/main/scala/scala/meta/Trees.scala) helpful to find out, what nodes I can match against.

Other than that, I start by writing my matchers and placing plenty of `println`s inside, to orient myself (and see how wrong I got things).

## A Concrete Example

Let's start with a syntactic rule.
[Tapir's DecodeFailureHandler](https://github.com/softwaremill/tapir/releases/tag/v1.9.0#DecodeFailureHandler) has changed.

What used to be

```scala
trait DecodeFailureHandler {
  def apply(ctx: DecodeFailureContext): Option[ValuedEndpointOutput[_]]
}
```

got a type argument for an effect type:

```scala
trait DecodeFailureHandler[F[_]] {
  def apply(ctx: DecodeFailureContext)(implicit monad: MonadError[F]): F[Option[ValuedEndpointOutput[_]]]
}
```

So I took a look at our actual DecodeFailureHandlers and made this test input:

```scala
/*
rule = FailureHandler
*/
import sttp.tapir.server.interceptor.decodefailure.{DecodeFailureHandler, DefaultDecodeFailureHandler}
import sttp.tapir.server.model.ValuedEndpointOutput
import sttp.tapir.server.interceptor.DecodeFailureContext
import scala.concurrent.ExecutionContext

class FailureHandler extends DecodeFailureHandler {

  private def renderResponse(): ValuedEndpointOutput[_] = ???

  override def apply(ctx: DecodeFailureContext): Option[ValuedEndpointOutput[_]] = {
      DefaultDecodeFailureHandler.respond(ctx) match {
        case Some((sc, hs)) =>
          Some(renderResponse())
        case None =>
          None
      }
    }
}

class DifferentClass {
  // Dummy to keep EC import
  def bar(ec: ExecutionContext) = ???

  // Don't rewrite this
  def apply(ctx: DecodeFailureContext): Option[ValuedEndpointOutput[_]] = {
    ???
  }
}
```

As this rule is intended for our services using Play, we'll use `Future` as our effect type.
The second class was added, to make sure my rule wouldn't rewrite similar looking code. The `ExecutionContext` import was added,
because my first draft messed up some code which already had the EC import.

And what is our desired output? We need some imports (potentially), substitute F with Future, add the implicit MonadError and wrap the result in a Future:

```scala
import sttp.tapir.server.interceptor.decodefailure.{DecodeFailureHandler, DefaultDecodeFailureHandler}
import sttp.tapir.server.model.ValuedEndpointOutput
import sttp.tapir.server.interceptor.DecodeFailureContext
import scala.concurrent.ExecutionContext
import scala.concurrent.Future
import sttp.monad.MonadError

class FailureHandler()(implicit ec: ExecutionContext) extends DecodeFailureHandler[Future] {

  private def renderResponse(): ValuedEndpointOutput[_] = ???

  override def apply(ctx: DecodeFailureContext)(implicit monad: MonadError[Future]): Future[Option[ValuedEndpointOutput[_]]] = Future {
      DefaultDecodeFailureHandler.respond(ctx) match {
        case Some((sc, hs)) =>
          Some(renderResponse())
        case None =>
          None
      }
    }
}

class DifferentClass {
  // Dummy to keep EC import
  def bar(ec: ExecutionContext) = ???

  // Don't rewrite this
  def apply(ctx: DecodeFailureContext): Option[ValuedEndpointOutput[_]] = {
    ???
  }
}
```

Great. Let's start with the rule:

```scala
package v2_5

import scalafix.v1._
import scala.meta._

class FailureHandler extends SyntacticRule("FailureHandler") {
  override def isRewrite: Boolean = true
  override def description: String = "You can add a description, if you like"
  override def fix(implicit doc: SyntacticDocument): Patch = ???
}
```

The `fix` method is where the magic happens.
As a starting point, we want to match on a class definition, which extends `sttp.tapir.server.interceptor.decodefailure.DecodeFailureHandler`.
I'm cutting some corners here. Syntactic rules are simpler, as mentioned above. I am able to grep our entire code base,
so I know, that we don't have any other `DecodeFailureHandler` this one could be confused with.
This is actually a very specific scenario and I know very well which files this should affect.
So I can get away with a syntactic rule, just matching on "some class" extending something named "DecodeFailureHandler".
You could use scalafix' [Symbol](https://scalacenter.github.io/scalafix/docs/developers/symbol-information.html#lookup-class-parents) to lookup a class and its parents. But that needs semantic information.

Let's match on a class definition:

```scala
override def fix(implicit doc: SemanticDocument): Patch = {
    val clazz = doc.tree.collect {
      case c @ Defn.Class.After_4_6_0(
            _,
            name,
            params,
            Ctor.Primary.After_4_6_0(_, _, ctorParams),
            Template.After_4_4_0(_,
                                 List(Init.After_4_6_0(Type.Name("DecodeFailureHandler"), Name.Anonymous(), Nil)),
                                 _,
                                 stats,
                                 _)) if params.values.isEmpty && ctorParams.isEmpty => ???
        // More to come...
  }
  ???
}
```

Note that the `fix` method returns a [Patch](https://scalacenter.github.io/scalafix/docs/developers/patch.html), which represents the changes we are performing. This type is very important, you'll need those methods to generate the changes you want.

Inside the method, we apply our partial function to the syntax tree with `collect`.
The `Defn.Class` node represents a class definition. What about the ugly `After_4_6_0`? Scalameta uses these versioned matchers for backwards compatibility. This is a great thing, otherwise every new version might break your rules.

The class definition contains a bunch of stuff: `Defn.Class(modifiers, name, paramClause, ctor, template)`.
This `Template` structure is very confusing, but contains lots of things we care about. Looking at this in the [AST explorer](https://astexplorer.net/#/gist/ec56167ffafb20cbd8d68f24a37043a9/24f036976d6f7869e4b4cc74bd835c43093b8c2f) brings some clarity.

Note that this matches syntactically, i.e., this matches only if the class only extends `DecodeFailureHandler`. If you test for multiple super-classes, you'll have to search the list of `Inits` (or use a semantic rule).
I'm also testing for "no type parameters" and "empty constructor", because that's what I see in our code base. Your mileage may vary.

Now I want to modify the class definition, by adding an implicit argument and by adding `[Future]` to `DecodeFailureHandler` and by wrapping `Future.apply` around the method body. I know that all source files I want to rewrite use curly braces around the body, so I can get away with simply placing `Future` before the opening brace `{`.
This means I want a patch that manipulates the tokens of the tree node we just matched:

```scala
// continued in the match above
val tokens     = c.tokens  // Tokens of the class definition
val identifier = tokens.find(_.is[Token.Ident]) // First identifier should be the name of declared class
val traitName  = tokens.find { case Token.Ident("DecodeFailureHandler") => true; case _ => false } // Find the token in the `extends` clause
```

Patch lets us add text left or right of tokens, replace tree nodes, remove tokens, etc.
So given `FailureHandler` (remember, no type parameters, no constructors), we want `FailureHandler()(implicit ec: ExecutionContext)`. We can just replace the identifier token: `Patch.replaceToken(identifier.get, s"${name.value}()(implicit ec: ExecutionContext)")`[^2]. Yes, we're playing fast and loose with Options here, but I'm sure that token exists, since we matched the tree node.

Adding the type argument to `DecodeFailureHandler` is even easier: `Patch.addRight(traitName.get, "[Future]")`.

How do we get to the `apply` method? See this `stats` field in the class definitions template above?
These are the _statements_ that make up the body of the class.
We need to match on those statements, to find the right method definition:

```scala
  stats.collect { case m @ Defn.Def.After_4_7_3(mods, Term.Name("apply"), _, Some(tpe), body) =>
    val modsStr = mods.mkString(" ")
    Patch.replaceTree(
      m,
      s"$modsStr def apply(ctx: DecodeFailureContext)(implicit monad: MonadError[Future]): Future[$tpe] = Future $body")
  }.asPatch
```

Again, I can simplify things a bit, because I know what the code looks like, on which this rule will be applied.
I'm simply replacing the whole tree node, instead of fiddling around with tokens.

These three patches need to be assembled into one, e.g., like so: `Patch.fromIterable(Seq(p1, p2, p3))`.

The last piece missing is the imports. I know that some of the sources have some of the required imports, so I don't want to add them unconditionally (although this could be cleaned up with another scalafix rule, `RemoveUnused`), but the `MonadError` import is never present:

```scala
  val imports = Patch.addGlobalImport(importer"sttp.monad.MonadError") +
    (if (hasImport(doc.tree, "Future")) Patch.empty
      else Patch.addGlobalImport(importer"scala.concurrent.Future")) +
    (if (hasImport(doc.tree, "ExecutionContext")) Patch.empty
      else Patch.addGlobalImport(importer"scala.concurrent.ExecutionContext"))
```

The `hasImport` method just matches on the tree, looking for `Import` statements.

Since we only want to add the imports, if this file _actually contains_ a failure handler, we assemble the final patch conditionally:

`if (clazz.isEmpty) Patch.empty else (imports + clazz)`

The complete rule can be found in this [gist](https://gist.github.com/cptwunderlich/8cbb9ae09b0d7cabdcd4a8b72183c363).

## Semantic Rules

Additionally to what you can do in a syntactic rule, you may also use [symbol matchers](https://scalacenter.github.io/scalafix/docs/developers/symbol-matcher.html) and so on.

You can use a symbol matcher in a pattern matching a tree node. For example, if we want to match an element in a parameter list of type `play.api.ApplicationLoader.Context`, we can do this like so:

```scala
  private def patchContextParam(paramClause: Term.ParamClause)(implicit doc: SemanticDocument) = {
    val contextMatcher = SymbolMatcher.exact("play/api/ApplicationLoader.Context#")

    paramClause.values.collect { case p @ Term.Param(_, name, Some(contextMatcher(tpe)), _) =>
      Patch.replaceTree(p, s"$name: $tpe")
    }.asPatch
  }
```

We place the matcher `contextMatcher(_)` where want to match a tree node in our match expression.
The name in parenthesis is bound to the matched tree node.

## Applying Rules

This is where things got complicated for me.

### Private Repos

As outlined in the beginning, we need to publish rules to a private repository.
This simply uses our normal release process. It just pushes some jar files to Nexus.

When _consuming_ rules from a private repository though, you need to add a _resolver_.
We have an sbt resolver configured for our normal dependencies, but we need an additional *scalafixResolver*.

If you want to use the rules from your sbt project, add the resolver to the build:

`ThisBuild / scalafixResolvers += coursierapi.MavenRepository.of("https://private.reposit.ory")`

Since I created these migrations for use with scala-steward, this is not necessary (I will explain scala-stewards mechanism soon).
But it is useful for testing and applying rules manually.
You can also add it to the current sbt session only, by typing `set ThisBuild / scalafixResolvers += coursierapi.MavenRepository.of("https://private.reposit.ory")` in the sbt console.

### Sbt vs. scalafix-cli

When run from the sbt build, the scalafix rules can't rewrite the sbt build files themselves. Like that `Dependency.scala` file I wanted to rewrite, remember? At least, I haven't found a way to achieve this.

But there is a [scalafix CLI tool](https://scalacenter.github.io/scalafix/docs/users/installation.html#command-line), which can be applied to any file.

This is especially useful for standard rules and those published via http/GitHub. To use artifacts from a private repository, you need to resolve them yourself. Luckily, that is quite simple with coursier:

```
scalafix \
  --tool-classpath $(cs fetch my.org::my-scalafix-rules:2.5.0 -p) \
  -r TapirPlay29Dependencies \
  ~/myproject/project/Dependencies.scala
```

You may also want to use the `--scala-version` flag. I wrote and published my rules in Scala 2.13, which is the current default.

One *important thing* to note is the double semicolon `::` in the artifact coordinates. This works similar to sbt's `%%` for cross-compiling. Many of the existing rules and examples use only one semicolon. Apparently, this is from an older version, which didn't support cross-publishing. You might have to add the version suffix ("_2.13") if you use that.

## Scala-Steward

The star of the show! Let's put everything together to realize my dream of _automagic_ upgrades!

### Try Before You Buy!

The good thing about reading blog posts about things is, that you don't have to go through the mistakes the author has gone through in order to write this.

When scala-steward opens a PR on GitHub - that's it! If you mess up the config, or your rules don't work quite like you wanted, you don't get a second chance. You can't delete PRs, only close them. Scala-steward will see a closed PR for the given version and that's that.

So to test your config, you can issue updates against a different branch from _main_.
To make testing simpler, I used the scala-steward docker container.

Here is the magical incantation:

```
docker run -v /home/ben/Projects/scala-steward-test:/opt/scala-steward \
  -v /home/ben/.sbt:/root/.sbt \
  -v /home/ben/.config/coursier:/root/.config/coursier \
  --env COURSIER_REPOSITORIES="ivy2Local|central|https://my.private.repo" \
  -it fthomas/scala-steward:latest \
  --workspace  "/opt/scala-steward/workspace" \
  --git-author-email "benjamin.maurer@noreply.com" \
  --do-not-fork \
  --forge-login "scala-steward-test" \
  --repos-file "/opt/scala-steward/repos.md" \
  --repo-config "/opt/scala-steward/default.scala-steward.conf" \
  --git-ask-pass "/opt/scala-steward/.github/askpass/pass.sh" \
  --scalafix-migrations "/opt/scala-steward/scalafix-migrations.conf"
```

What does this mean?
I'm mounting the directory `/home/ben/Projects/scala-steward-test` as `/opt/scala-steward` in the container. I have all the configuration files in this folder.
The folders `~/.sbt` and `~/.config/coursier` are mounted into the container, so it has my credentials for the private repository (this is not a very secure way of doing things, but for a quick test, it's OK).
The environment variable `COURSIER_REPOSITORIES` tells coursier which repositories to use to resolve artifacts.
My `pass.sh` is a very janky construct, that just echos my personal [GitHub access token](https://github.com/settings/tokens).

`repos.md` contains the repository configuration. It's a markdown list in the format `- org/repo[:branch]`.
If you don't specify a branch, it will use main/master.

`default.scala-steward.conf` the, well, default scala-steward config (can be overwritten with project specific configs).
You might want to allow pre-release updates for your artifact, if you are testing migrations before the release:
`updates.allowPreReleases = [ { groupId = "my.org", artifactId = "my-artifact" } ]`

`scalafix-migrations.conf` contains the list of migrations.
I found the [documentation](https://github.com/scala-steward-org/scala-steward/blob/main/docs/scalafix-migrations.md) of this a bit lacking, but you can take a [look at the sources](https://github.com/scala-steward-org/scala-steward/blob/main/modules/core/src/main/scala/org/scalasteward/core/edit/scalafix/ScalafixMigration.scala#L28) to find out what all the fields mean (I'm planning on improving the docs).

Here an example:

```
migrations = [
  {
    groupId: "my.org",
    artifactIds: ["my-artifact"],
    newVersion: "2.5.0-RC1",
    rewriteRules: [
        "dependency:FailureHandler@my.org::my-scalafix-rules:2.5.0-RC7",
        "replace:akka.persistence.jdbc.config.JournalTableConfiguration/akka.persistence.jdbc.config.LegacyJournalTableConfiguration"
    ],
    executionOrder: "post-update",
    target: "sources",
    doc: "https://cptwunderlich.github.io/"
  }
]
```

First you define for _which_ artifact and version the migration applies. `rewriteRules` contains a list of scalafix rules, with a protocol prefix (http, github, dependency...), unless it is a built-in rule.

Now it gets interesting. `executionOrder` can be `pre-update`, i.e., before the artifact version is incremented, or `post-update`.
According to the docs, a `pre-update` migration could affect the update, so it is resolved again (i.e., you may change `Dependencies.scala`).

`target` can be `sources` or `build`. This one's interesting and not well documented. Basically, "sources" is a transformation of the project's sources, just like the sbt-based scalafix invocation seen earlier. "build" can be applied to the build files.
And if we look at the implementation, this actually maps to the sbt-scalafix vs scalafix-cli scenario above.
There is a difference though. "source" creates a new sbt project and sets the `scalafixResolvers` from the coursier resolvers. That's why you don't need to set the `scalafixResolver` in your project for scala-steward - and on the flipside only setting it there won't make it work in scala-steward.

This brings me to my first big defeat: the "build" variant invokes scalafix-cli, which can't resolve artifacts from private repositories on its own. The trick above with downloading dependencies with coursier and using `--tools-classpath` doesn't work, because scala-steward can't do this.
For now, this just doesn't work and there is no way to run a shell script or similar. (But maybe we can make this work [#3276](https://github.com/scala-steward-org/scala-steward/issues/3276), [#3133](https://github.com/scala-steward-org/scala-steward/pull/3133)).
I also tried using an artifact migration for this, but this seems to only be triggered if there is a new version for the artifact to be migrated. It didn't work either way.

### Github Action

I added the `scalafix-migration.conf` to the root directory of our GitHub repo with the config for the [scala-steward GitHub action](https://github.com/scala-steward-org/scala-steward-action).

To the job's steps, I added the checkout action and then I can simply refer to the config file:

```yaml
jobs:
  scala-steward:
    runs-on: ubuntu-22.04
    name: Launch Scala Steward
    steps:
      - name: Add SBT credentials for Nexus
        run: | # redacted

      - name: Checkout repo so migrations config is available
        uses: actions/checkout@v4
      - name: Launch Scala Steward
        uses: scala-steward-org/scala-steward-action@v2
        with:
          github-app-id: ${{ secrets.SCALA_STEWARD_APP_ID }}
          github-app-installation-id: ${{ secrets.SCALA_STEWARD_INSTALLATION_ID }}
          github-app-key: ${{ secrets.SCALA_STEWARD_PRIVATE_KEY }}
          scalafix-migrations: 'scalafix-migrations.conf'
```

## Conclusion

So, did I achieve an automatic upgrade? Far from it.
Scalafix and scala-steward are amazing tools and I'm happy I invested the time. I'm sure this will make future migrations easier.

### Limitations

There are a few limitations though:

1. I didn't mention it so far, but you can't change configuration files this way.
2. Migrations of build files don't work with private repos - you'll need to publish to maven central, or put the source on a public GitHub repo or web site.
3. Migrations can fail if the code doesn't compile, or the semanticDB is stale (and the PR will be opened nonetheless [#3273](https://github.com/scala-steward-org/scala-steward/issues/3273)).

I'm not quite sure I understand nr. 3 completely. After upgrading the dependency, the code doesn't compile anymore. Syntactic rules still work, bc. those are not parser errors, but type errors.
It seems like semantic rules don't work on sources that don't compile (yet). Maybe bc. scala-steward checks the project out and doesn't have old semanticDBs lying around?
Speaking of which, some users report the error "stale semanticDB found". This might be the case, if one rule changes the code, but it is not re-compiled - not quite sure.

In general have I seen issue when mixing multiple semantic rules. Sometimes it seems like the first rule works, but then the code doesn't compile properly so the second rule doesn't. This is just anecdotal, I have not investigated this. But the built-in `replace` rule seems never to work for me in combination with other rules. I just went back to using `sed` where possible.

### Documentation

I think the docs can be improved. I had quite the hard start with scalafix. But it might be me.
Now in hindsight, the docs make a lot of sense to me. But in the beginning, I didn't quite get it and more importantly, I couldn't find a lot of things.

A few words on cross-compilation, or rather which Scala versions to use might be helpful. I wasn't sure whether rules had to be compiled with the same version as the code they are applied to (seems like they don't).
I also at first thought, that they require Scala 2.12, like sbt does. But the rules I looked at seem to be old.
Scala 2.13 is the default, AFAIK.

The scala-steward documentation lacks a description of the config values for migrations.
Private repos could also do with some better explanations.

I hope I can contribute this sometime soon.

If you start out with scalafix (and scala-steward), don't make the same mistake I did - take more notes on which things caused you problems, or where the docs were not clear enough. That will help to improve them!

### PEBKAC

Remember those double semicolons? They are important!
After all the testing and releasing libraries and preparatory work, I messed up the scala-steward config in a flurry.
I had single semicolons and scala-steward tried to resolve the wrong version.
So instead of half-applied migrations, I got none...

My colleagues had to make due with a migration guide, containing lots of `sed` commands and big `scalafix` incantation.

[^1]: I won't forget, I won't forgive. /jk
[^2]: Can't remember why I didn't just add the text right of the token, but I probably tried that. ¯\_(ツ)_/¯

