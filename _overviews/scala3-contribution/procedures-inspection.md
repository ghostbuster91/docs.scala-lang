---
title: How to Inspect Values
type: section
description: This page describes inspecting semantic values in the Scala 3 compiler.
num: 8
previous-page: procedures-areas
next-page: procedures-efficiency
---

In this section, we take a closer look at how to debug the contents of certain objects
in the compiler, and produced artifacts.

## Inspecting variables in-place

Frequently we need to inspect the content of a particular variable. Often, it is sufficient to use `println`.

When printing a variable, it's always a good idea to call `show` on that variable: `println(x.show)`.
Many objects of the compiler define `show`, returning a human-readable string.
e.g. if called on a tree, the output will be the tree's representation as source code, rather than
the raw data underlying.

Sometimes you need to print flags. Flags are metadata attached to [symbols] containing information such as whether a
class is abstract, comes from Java, what modifiers a variable has (private, protected etc) and so on.
Flags are stored in a single `Long` value, each bit of which represents whether a particular flag is set.

To print flags, you can use the `flagsString` method, e.g. `println(x.flagsString)`.

## Pretty Printing with a String Interpolator

You can also pretty print objects with string interpolators,
these default to call `.show` when possible, avoiding boilerplate
and also helping format error messages.

Import them with the following:

```scala
import dotty.tools.dotc.core.Decorators.*
```

Here is a table of explanations for their use:

| Usage  | Description                       |
|--------|-----------------------------------|
|`i""`   | General purpose string formatting. It calls `.show` on objects <br/> mixing in Showable, `String.valueOf` otherwise |
|`em""`  | Formatting for error messages: Like `i` but suppress <br/>follow-on, error messages after the first one if some <br/>of their arguments are "non-sensical". |
|`ex""`  | Formatting with added explanations: Like `em`, but add <br/>explanations to give more info about type variables<br/>and to disambiguate where needed. |


## Obtaining debug output from the compiler

There are many compiler options that provide verbose debug output when compiling a file.
You can find the full list in [ScalaSettings.scala] file. A particularly useful one
is `-Xprint:<phase-name>` or `-Xprint:all`. It prints trees after a given phase or after
all phases. As described in the [compiler lifecycle][3] each phase transforms the trees
and types that represent your code in a certain way. This flag allows you to see exactly how.

## Stopping the compiler early
Sometimes you may want to stop the compiler after a certain phase, for example to prevent
knock-on errors from occurring from a bug in an earlier phase. Use the flag
`-Ystop-after:<phase-name>` to prevent any phases executing afterwards.

> e.g. `-Xprint:<phase>` where `phase` is a miniphase, will print after
> the whole phase group is complete, which may several miniphases after `phase`.
> Instead you can use `-Ystop-after:<phase> -Xprint:<phase>` to stop
> immediately after the miniphase and see the trees that you intended.

## Printing TASTy of a Class

If are working on an issue related to TASTy, it is good to know how to inspect
the contents of a TASTy file, produced from compilation of Scala files.

In the following example, we compile in an [issue directory][reproduce] `tasty/Foo.scala`,
with contents of `class Foo`, and create a `tasty/launch.iss` file to print its TASTy
with sbt command `issue tasty`:

```
$ (rm -rv out || true) && mkdir out # clean up compiler output, create `out` dir.

scala3/scalac -d $here/out $here/Foo.scala

scala3/scalac -print-tasty $here/out/Foo.tasty
```

We see output such as the following:

```
--------------------------------------------------------------------------------
local/foo/out/Foo.tasty
--------------------------------------------------------------------------------
Names:
   0: ASTs
   1: <empty>
   2: Foo
   3: <init>
...
```
and so on.

## Inspecting representation of types

> [click here][2] to learn more about types in `dotc`.

If you are curious about the representation of a type, say `[T] =>> List[T]`,
you can use a helper program [dotty.tools.printTypes][1],
it prints the internal representation of types, along with their class. It can be
invoked from the sbt shell with three arguments as follows:
```bash
sbt:scala3> scala3-compiler/Test/runMain
  dotty.tools.printTypes
  <source>
  <kind>
  <typeStrings*>
```

- The first argument, `source`, is an arbitrary string that introduces some Scala definitions.
It may be the empty string `""`.
- The second argument, `kind`, determines the format of the following arguments,
accepting one of the following options:
  - `rhs` - accept return types of definitions
  - `class` - accept signatures for classes
  - `method` - accept signatures for methods
  - `type` - accept signatures for type definitions
  - The empty string `""`, in which case `rhs` will be assumed.
- The remaining arguments are type signature strings, accepted in the format determined by
`kind`, and collected into a sequence `typeStrings`. Signatures are the part of a definition
that comes after its name, (or a simple type in the case of `rhs`) and may reference
definitions introduced by the `source` argument.

Each one of `typeStrings` is then printed, displaying their internal structure, alongside their class.

### Examples

Here, given a previously defined `class Box { type X }`, we inspect the return type `Box#X`:
```bash
sbt:scala3> scala3-compiler/Test/runMain
> dotty.tools.printTypes
> "class Box { type X }"
> "rhs"
> "Box#X"
[info] running (fork) dotty.tools.printTypes "class Box { type X }" rhs Box#X
TypeRef(TypeRef(ThisType(TypeRef(NoPrefix,module class <empty>)),class Box),type X) [class dotty.tools.dotc.core.Types$CachedTypeRef]
```

Here are some other examples you can follow:
- `...printTypes "" "class" "[T] extends Seq[T] {}"`
- `...printTypes "" "method" "(x: Int): x.type"`
- `...printTypes "" "type" "<: Int" "= [T] =>> List[T]"`

### Don't just print: extracting further information

`dotty.tools.printTypes` is useful to to at a glance see the representation
of a type, but sometimes we want to extract more. We can instead use the
method `dotty.tools.DottyTypeStealer.stealType`. With the same inputs as `printTypes`,
it returns both a `Context` containing the definitions passed, along with the list of types.

As a worked example let's create a test case to verify the structure of `Box#X` that we saw earlier:
```scala
import dotty.tools.dotc.core.Contexts.Context
import dotty.tools.dotc.core.Types.*

import org.junit.Test

import dotty.tools.DottyTypeStealer, DottyTypeStealer.Kind

class StealBox:

  @Test
  def stealBox: Unit =
    val (ictx, List(rhs)) =
      DottyTypeStealer.stealType("class Box { type X }", Kind.rhs, "Box#X")

    given Context = ictx

    rhs match
      case X @ TypeRef(Box @ TypeRef(ThisType(empty), _), _) =>
        assert(Box.name.toString == "Box")
        assert(X.name.toString == "X")
        assert(empty.name.toString == "<empty>")
```

[1]: https://github.com/lampepfl/dotty/blob/master/compiler/test/dotty/tools/DottyTypeStealer.scala
[2]: {% link _overviews/scala3-contribution/arch-types.md %}
[3]: {% link _overviews/scala3-contribution/arch-lifecycle.md %}#phases
[ScalaSettings.scala]: https://github.com/lampepfl/dotty/blob/master/compiler/src/dotty/tools/dotc/config/ScalaSettings.scala
[symbols]: https://github.com/lampepfl/dotty/blob/master/compiler/src/dotty/tools/dotc/core/SymDenotations.scala
[reproduce]: {% link _overviews/scala3-contribution/procedures-reproduce.md %}#dotty-issue-workspace
