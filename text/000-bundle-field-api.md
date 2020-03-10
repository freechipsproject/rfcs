- 000 Bundle Field API
- Start Date: 2020-03-10
- RFC PR: (leave this empty)
- Chisel-lang Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes add `Field` as an extension to the `Bundle` construction API to specify
precisely those parts of a `Bundle` that are actually intended to be functioning fields

# Motivation
[motivation]: #motivation

The main goal of this addition is to further reduce the number of situations
where a developer needs to write a `cloneType` method for custom `Bundle`s.
It is further motivated by the difficulty in recognizing problematic constructions and
crafting comprehensible error messages when they occur.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

It is proposed that an optional binding operator `Field` be added to the `Bundle` construction API.
`Field` is similar to the binding operators `IO`, `Input`, `Output`, etc.
It will allow with better semantics, which will coexist with the old semantics indefinitely. 

Simple example:
```scala
class MyBundle extends Bundle {
  val x = UInt(32.W)
  val y = UInt(16.W)
  val flag = Bool()
}
```
would optionally become.
```scala
class MyBundle extends Bundle {
  val x = Field(UInt(32.W))
  val y = Field(UInt(16.W))
  val flag = Field(Bool())
}
```

Notable in this the following, `gen` and `constant` are not treated as fields of `MyFancierBundle`

```scala
class MyFancierBundle(val gen: Data) extends Bundle {
  val x = Field(UInt(32.W))
  val y = Field(UInt(16.W))
  val flag = Field(Bool())
  val constant = 7.U
}
```

The new semantics are, tentatively:

`Fields` are all or nothing, if you define any `Bundle` element as a `Field`, you must define all Bundle elements as Fields. The compiler will error out if you either have vals of type Data that aren't declared as a Field, or if you have Fields that weren't autodetected to be Bundle fields (which is a limitation of reflection-based detection).
As it's not possible to have a Field(...) declaration in a constructor argument, constructor Data vals will NOT be treated as fields.
`Field` will be a Bundle-specific API, and cannot be called outside a Bundle.
This should eliminate all uses of cloneType in non-experimental Chisel constructs.

In experimental, Record may still need cloneType, but Record is an advanced construct, and we may redefine it in the future to be more friendly.
There was discussion about changing the name of this new API, for example as Struct (or something) instead of Bundle. I'm inclined to stick with Bundle, but there may be a discussion to be continued here.
We will encourage the use of Field in new code, and there is no set timeline for when (if ever) old style code will generate deprecations, warnings, or errors.

Suggestions are welcome!

# Background
[background]: #background

The standard pattern for using Chisel Data objects as generator parameters in `Bundle`s is to pass 
them as a constructor argument like so:
```scala
class MyBundle[T <: Data](gen: T) extends Bundle {
  val foo = Input(gen)
}
```
In the modern world of autoclonetype; however, you are encouraged to make constructor 
arguments `val`s so that reflection can find them. If you make `gen` a `val`:
```scala
class MyBundle[T <: Data](val gen: T) extends Bundle {
  val foo = Input(gen)
}
```
Whoops, you've accidentally added a new element to your `Bundle` called `gen`!
This is due to the fact that the elements of a `Bundle` are defined (by implementation) to 
be the public vals of type `Data` or `Some[Data]`.

Instead, you should use a private val:
```scala
class MyBundle[T <: Data](private val gen: T) extends Bundle {
  val foo = Input(gen)
}
```
While a little verbose this allows autoclonetype to automatically generate clonetype so everyone is happy.

This led to a discussion on what the fundamental definition of a Bundle is/should be.
As far as I can tell there are two possibilities:

### 1. Bundle Elements are the public `val`s of the Scala class (the status quo)

As far as I know, this hasn't been much a point of confusion for new users; however,
with autoclonetype encouraging people to make their constructor parameters `vals` it most certainly could become one.
To make this better, I think there are a couple of things we should do:
1. Write a wiki page on the definition of a Bundle with examples
1. When a `chisel3.core.Binding$MixedDirectionAggregateException` caused by a val constructor parameter of type `Data` or `Some[Data]`, inform the user that it counts as an element and link to the wiki page.
1. Augment autoclonetype to detect elements that are also constructor parameters and implicitly call `cloneType` on them. This is needed since these constructor arguments are not just generator parameters, they belong to the Bundle so we need fresh objects.
    * Added bonus of this is it should enable single-line case class Bundles

There is one case where I'm not sure if we'll be able to [easily] provide a decent error message though. 
For a Bundle with no directions at all, you can end up with a 
`firrtl.passes.CheckInitialization$RefNotInitializedException` if you have a val constructor 
parameter of type `Data` that you didn't intend to be an element of the Bundle.

### 2. Bundle Elements should be vals in the *body* of a Bundle
There are very few examples of people using val constructor parameters to create fields of Bundles.
One example: [here](https://github.com/ucb-bar/dsptools/blob/master/src/main/scala/dsptools/numbers/chisel_concrete/DspComplex.scala#L59) h/t @grebe). 

Perhaps it is better to redefine `Bundle`s such that constructor arguments of type `Data` or 
`Some[Data]` are not elements, rather they are just constructor parameters.
This has the nice property of encouraging separation of generator parameters and elements of the `Bundle`, 
and encourages people to express `Bundle`s in a way similar to a C-struct. 
Misuse (eg. connecting to a val that is a constructor parameter) is also very easy to detect
and provide a specific error message. As far as I know there aren't any obvious cases we can't catch like the one above.
That being said this kind of a change would require carefully considering its implications on inheritance 
(especially overriding elements that may be a constructor parameters for a parent class)

# Drawbacks
[drawbacks]: #drawbacks

This creates more boiler plate code, that most of the time seems unnecessary.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Strong deterministic identification of elements intended to be fields of a `Bundle` should help developers
- Limiting user created 'cloneType' methods should help and developers and chisel support personnel.


# Prior art
[prior-art]: #prior-art

[Chisel3 PR #909 Field](https://github.com/freechipsproject/chisel3/pull/909) was an earlier attempt to 
implement the `Field` API extension. Other related issues (used in constructing this RFC) are:

- [What does it mean to be a Bundle? #765](https://github.com/freechipsproject/chisel3/issues/765)
- [Field (see above)](https://github.com/freechipsproject/chisel3/pull/909)
- [Add test for behavior of bundles with cloneType](https://github.com/freechipsproject/chisel3/pull/908)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How will the community feel about this API

# Future possibilities
[future-possibilities]: #future-possibilities

There maybe other alternatives like `macro`s, compile plug-ins, perhaps other techniques 
that would make solve the target issues here with less impact on the developers
