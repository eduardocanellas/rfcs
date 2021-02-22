- Feature Name: try_trait_v2
- Start Date: 2020-12-12
- RFC PR: [rust-lang/rfcs#3058](https://github.com/rust-lang/rfcs/pull/3058)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Replace [RFC #1859, `try_trait`](https://rust-lang.github.io/rfcs/1859-try-trait.html),
with a new design for the currently-unstable [`EarlyExit` trait](https://doc.rust-lang.org/nightly/std/ops/trait.EarlyExit.html)
and corresponding desugaring for the `?` operator.

The new design supports all the currently-stable conversions (including the accidental ones),
while addressing the discovered shortcomings of the currently-implemented solution,
as well as enabling new scenarios.

*This is forward-looking to be compatible with other features,
like [`try {}`](https://doc.rust-lang.org/nightly/unstable-book/language-features/try-blocks.html) blocks
or [`yeet e`](https://twitter.com/josh_triplett/status/1248658754976927750) expressions
or [`Iterator::try_find`](https://github.com/rust-lang/rust/issues/63178),
but the statuses of those features are **not** themselves impacted by this RFC.*

# Motivation
[motivation]: #motivation

The motivations from the previous RFC still apply (supporting more types, and restricted interconversion).
However, new information has come in since the previous RFC, making people wish for a different approach.

- Using the "error" terminology is a poor fit for other potential implementations of the trait.
- The ecosystem has started to see error types which are `From`-convertible from *any* type implementing `Debug`, which makes the previous RFC's method for controlling interconversions ineffective.
- It's no longer clear that `From` should be part of the `?` desugaring for _all_ types.  It's both more flexible -- making inference difficult -- and more restrictive -- especially without specialization -- than is always desired.
- An [experience report](https://github.com/rust-lang/rust/issues/42327#issuecomment-366840247) in the tracking issue mentioned that it's annoying to need to make a residual type in common cases.

This RFC proposes a solution that _mixes_ the two major options considered last time.

- Like the _reductionist_ approach, this RFC proposes an unparameterized trait with an _associated_ type for the "ok" part, so that the type produced from the `?` operator on a value is always the same.
- Like the [_essentialist_ approach](https://github.com/rust-lang/rfcs/blob/master/text/1859-try-trait.md#the-essentialist-approach), this RFC proposes a trait with a _generic_ parameter for "error" part, so that different types can be consumed.

<!--
Why are we doing this? What use cases does it support? What is the expected outcome?
-->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The `ops::ControlFlow` type

This is a simple enum:
```rust
enum ControlFlow<B, C = ()> {
    /// Exit the operation without running subsequent phases.
    Break(B),
    /// Move on to the next phase of the operation as normal.
    Continue(C),
}
```

It's intended for exposing things (like graph traversals or visitor) where you want the user to be able to choose whether to exit early.  Using an enum is clearer than just using a bool -- what did `false` mean again? -- as well as [allows it to carry a value](https://github.com/rust-lang/rust/pull/78779#pullrequestreview-524885131), if desired.

For example, you could use it to expose a simple tree traversal in a way that lets the caller exit early if they want:
```rust
impl<T> TreeNode<T> {
    fn traverse_inorder<B>(&self, mut f: impl FnMut(&T) -> ControlFlow<B>) -> ControlFlow<B> {
        if let Some(left) = &self.left {
            left.traverse_inorder(&mut f)?;
        }
        f(&self.value)?;
        if let Some(right) = &self.right {
            right.traverse_inorder(&mut f)?;
        }
        ControlFlow::Continue(())
    }
}
```
<!-- https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=286ac85c43e5bf242b5431b4f6f63386 -->


## The `EarlyExit` trait

The `ops::EarlyExit` trait describes a type's behavior when used with the `?` operator, like how the `ops::Add` trait describes its behavior when used with the `+` operator.

At its core, the `?` operator is about splitting a type into its two parts:

- The *remainder* that will be returned from the `?` expression, with which the program will continue, and
- The *return* that will be returned to the calling code, as an early exit from the normal flow.

(Oxford's definition for a residual is "a quantity remaining after other things have been subtracted or allowed for", thus the use here.)

The `EarlyExit` trait also has facilities for rebuilding a type from either of its parts.  This is needed to build the final return value from a function, both in `?` and in methods generic over multiple types implementing `EarlyExit`.

Here's a quick overview of a few standard types which implement `EarlyExit`, their corresponding output and residual types, and the functions which convert between them.
(Full details will come later; the goal for now is just to get the general idea.)

```text
+----------------------+                                 +-------------------------+                              +-----------------------+
| EarlyExit::Remainder |                                 |      EarlyExit Type     |                              |   EarlyExit::Return   |
+----------------------+   EarlyExit::branch is Continue +-------------------------+   EarlyExit::branch is Break +-----------------------+
|           T          |  <----------------------------  |      Result<T, E>       |  ------------------------->  |     Result<!, E>      |
|           T          |                                 |        Option<T>        |                              |       Option<!>       |
|           C          |  ---------------------------->  |    ControlFlow<B, C>    |  <-------------------------  |   ControlFlow<B, !>   |
+----------------------+      EarlyExit::from_remainder  +-------------------------+    EarlyExit::from_return    +-----------------------+
```

If you've used `?`-on-`Result` before, that output type is likely unsurprising.  Since it's given out directly from the operator, there's not much of a choice.

The residual types, however, are somewhat more interesting.  Code using `?` doesn't see them directly -- their usage is hidden inside the desugaring -- so there are more possibilities available.  So why are we using these ones specifically?

Most importantly, this gives each family of types (`Result`s, `Option`s, `ControlFlow`s) their own *distinct* residual type.  That avoids unrestricted *interconversion* between the different types, the ability to arbitrarily mix them in the same method.  For example, it was mentioned earlier that just because a `ControlFlow::Break` is also an early exit, that doesn't mean that it should be allowed to consider it a `Result::Err` -- it might be a success, conceptually.  So by giving `ControlFlow<X, _>` and `Result<_, X>` different residual types, it becomes a compilation error to use the `?` operator on a `ControlFlow` in a method which returns a `Result`.  (There are also ways to allow interconversion is specific situations where it's desirable.)

> 🏗️ Note for those familiar with the previous RFC 🏗️
>
> This is the most critical semantic difference.  Structurally this definition of the trait is very similar to the previous -- there's still a method splitting the type into a discriminated union between two associated types, and constructors to rebuild it from them.  But by keeping the "result-ness" or "option-ness" in the residual type, it gives extra control over interconversion that wasn't possible before.  The changes other than this are comparatively minor, typically either rearrangements to work with that or renamings to change the vocabulary used in the trait.

Using `!` is then just a convenient yet efficient way to create those residual types.  It's nice as a user, too, not to need to understand an additional type.  Just the same "it can't be that one" pattern that's also used in `EarlyExitFrom`, where for example `i32::try_from(10_u8)` gives a `Result<i32, !>`, since it's a widening conversion which cannot fail.  Note that there's nothing special going on with `!` here -- any uninhabited `enum` would work fine.


## How error conversion works

One thing [The Book mentions](https://doc.rust-lang.org/stable/book/ch09-02-recoverable-errors-with-result.html#a-shortcut-for-propagating-errors-the--operator),
if you recall, is that error values in `?` have `From::from` called on them, to convert from one error type to another.

The previous section actually lied to you slightly: there are *two* traits involved, not just one.  The `from_return` method is on `FromReturn`, which is generic so that the implementation on `Result` can add that extra conversion.  Specifically, the trait looks like this:

```rust
trait FromReturn<Return = <Self as EarlyExit>::Return> {
	fn from_return(r: Return) -> Self;
}
```

And while we're showing code, here's the exact definition of the `EarlyExit` trait:

```rust
trait EarlyExit: FromReturn {
	type Remainder;
	type Return;
	fn branch(self) -> ControlFlow<Self::Return, Self::Remainder>;
	fn from_remainder(o: Self::Remainder) -> Self;
}
```

The fact that it's a super-trait like that is why I don't feel bad about the slight lie: Every `T: EarlyExit` *always* has a `from_return` function from `T::Return` to `T`.  It's just that some types might offer more.

Here's how `Result` implements it to do error-conversions:
```rust
impl<T, E, F: From<E>> FromReturn<Result<!, E>> for Result<T, F> {
    fn from_return(x: Result<!, E>) -> Self {
        match x {
            Err(e) => Err(From::from(e)),
        }
    }
}
```

But `Option` doesn't need to do anything exciting, so just has a simple implementation, taking advantage of the default parameter:

```rust
impl<T> FromReturn for Option<T> {
    fn from_return(x: Self::Return) -> Self {
        match x {
            None => None,
        }
    }
}
```

In your own types, it's up to you to decide how much freedom is appropriate.  You can even enable interconversion by defining implementations from the residual types of other families if you'd like.  But just supporting your one residual type is ok too.

> 🏗️ Note for those familiar with the previous RFC 🏗️
>
> This is another notable difference: The `From::from` is up to the trait implementation, not part of the desugaring.


## Implementing `EarlyExit` for a non-generic type

The examples in the standard library are all generic, so serve as good examples of that, but non-generic implementations are also possible.

Suppose we're working on migrating some C code to Rust, and it's still using the common "zero is success; non-zero is an error" pattern.  Maybe we're using a simple type like this to stay ABI-compatible:
```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq)]
#[repr(transparent)]
pub struct ResultCode(pub i32);
impl ResultCode {
    const SUCCESS: Self = ResultCode(0);
}
```

We can implement `EarlyExit` for that type to simplify the code without changing the error model.

First, we'll need a residual type.  We can make this a simple newtype, and conveniently there's a type with a niche for exactly the value that this can't hold.  This is only used inside the desugaring, so we can leave it opaque -- nobody but us will need to create or inspect it.
```rust
use std::num::NonZeroI32;
pub struct ResultCodeReturn(NonZeroI32);
```

With that, it's straight-forward to implement the traits.  `NonZeroI32`'s constructor even does exactly the check we need in `EarlyExit::branch`:
```rust
impl EarlyExit for ResultCode {
    type Remainder = ();
    type Return = ResultCodeReturn;
    fn branch(self) -> ControlFlow<Self::Return> {
        match NonZeroI32::new(self.0) {
            Some(r) => ControlFlow::Break(ResultCodeReturn(r)),
            None => ControlFlow::Continue(()),
        }
    }
    fn from_remainder((): ()) -> Self {
        ResultCode::SUCCESS
    }
}

impl FromReturn for ResultCode {
    fn from_return(r: ResultCodeReturn) -> Self {
        ResultCode(r.0.into())
    }
}
```

Aside: As a nice bonus, the use of a `NonZero` type in the residual means that `<ResultCode as EarlyExit>::branch` [compiles down to a nop](https://rust.godbolt.org/z/GxeYax) on the current nightly.  Thanks, enum layout optimizations!

Now, this is all great for keeping the interface that the other unmigrated C code expects, and can even work in `no_std` if we want.  But it might also be nice to give other *Rust* code that uses it the option to convert things into a `Result` with a more detailed error.

For expository purposes, we'll use this error type:
```rust
#[derive(Debug, Clone)]
pub struct FancyError(String);
```

(A real one would probably be more complicated and have a better name, but this will work for what we need here -- it's bigger and needs non-core things to work.)

We can allow `?` on a `ResultCode` in a method returning `Result` with an implementation like this:
```rust
impl<T, E: From<FancyError>> FromReturn<ResultCodeReturn> for Result<T, E> {
    fn from_return(r: ResultCodeReturn) -> Self {
        Err(FancyError(format!("Something fancy about {} at {:?}", r.0, std::time::SystemTime::now())).into())
    }
}
```

*The split between different error strategies in this section is inspired by [`windows-rs`](https://github.com/microsoft/windows-rs), which has both [`ErrorCode`](https://microsoft.github.io/windows-docs-rs/doc/bindings/windows/struct.ErrorCode.html) -- a simple newtype over `u32` -- and [`Error`](https://microsoft.github.io/windows-docs-rs/doc/bindings/windows/struct.Error.html) -- a richer type that can capture a stack trace, has an `Error` trait implementation, and can carry additional debugging information -- where the former can be converted into the latter.*


## Using these traits in generic code

`Iterator::try_fold` has been stable to call (but not to implement) for a while now.  To illustrate the flow through the traits in this RFC, lets implement our own version.

As a reminder, an infallible version of a fold looks something like this:
```rust
fn simple_fold<A, T>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> A,
) -> A {
    for x in iter {
        accum = f(accum, x);
    }
    accum
}
```

So instead of `f` returning just an `A`, we'll need it to return some other type that produces an `A` in the "don't short circuit" path.  Conveniently, that's also the type we need to return from the function.

Let's add a new generic parameter `R` for that type, and bound it to the output type that we want:
```rust
fn simple_try_fold_1<A, T, R: EarlyExit<Remainder = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    todo!()
}
```

`EarlyExit` is also the trait we need to get the updated accumulator from `f`'s return value and return the result if we manage to get through the entire iterator:
```rust
fn simple_try_fold_2<A, T, R: EarlyExit<Remainder = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    for x in iter {
        let cf = f(accum, x).branch();
        match cf {
            ControlFlow::Continue(a) => accum = a,
            ControlFlow::Break(_) => todo!(),
        }
    }
    R::from_remainder(accum)
}
```

We'll also need `FromReturn::from_return` to turn the residual back into the original type.  But because it's a supertrait of `EarlyExit`, we don't need to mention it in the bounds.  All types which implement `EarlyExit` can always be recreated from their corresponding residual, so we'll just call it:
```rust
pub fn simple_try_fold_3<A, T, R: EarlyExit<Remainder = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    for x in iter {
        let cf = f(accum, x).branch();
        match cf {
            ControlFlow::Continue(a) => accum = a,
            ControlFlow::Break(r) => return R::from_return(r),
        }
    }
    R::from_remainder(accum)
}
```

But this "call `branch`, then `match` on it, and `return` if it was a `Break`" is exactly what happens inside the `?` operator.  So rather than do all this manually, we can just use `?` instead:
```rust
fn simple_try_fold<A, T, R: EarlyExit<Remainder = A>>(
    iter: impl Iterator<Item = T>,
    mut accum: A,
    mut f: impl FnMut(A, T) -> R,
) -> R {
    for x in iter {
        accum = f(accum, x)?;
    }
    R::from_remainder(accum)
}
```


<!--
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.
-->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## `ops::ControlFlow`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum ControlFlow<B, C = ()> {
    /// Exit the operation without running subsequent phases.
    Break(B),
    /// Move on to the next phase of the operation as normal.
    Continue(C),
}
```

## The traits

```rust
pub trait EarlyExit: FromReturn {
    /// The type of the value consumed or produced when not short-circuiting.
    type Remainder;

    /// A type that "colours" the short-circuit value so it can stay associated
    /// with the type constructor from which it came.
    type Return;

    /// Used in `try{}` blocks to wrap the result of the block.
    fn from_remainder(x: Self::Remainder) -> Self;

    /// Determine whether to short-circuit (by returning `ControlFlow::Break`)
    /// or continue executing (by returning `ControlFlow::Continue`).
    fn branch(self) -> ControlFlow<Self::Return, Self::Remainder>;
}

pub trait FromReturn<Return = <Self as EarlyExit>::Return> {
    /// Recreate the type implementing `EarlyExit` from a related residual
    fn from_return(x: Return) -> Self;
}
```

## Desugaring `?`

The previous desugaring of `x?` was

```rust
match EarlyExit::into_result(x) {
	Ok(v) => v,
	Err(e) => return EarlyExit::from_error(From::from(e)),
}
```

The new one is very similar:

```rust
match EarlyExit::branch(x) {
	ControlFlow::Continue(v) => v,
	ControlFlow::Break(r) => return FromReturn::from_return(r),
}
```

It's just left conversion (such as `From::from`) up to the implementation instead of forcing it in the desugar.

## Standard implementations

### `Result`

```rust
impl<T, E> ops::EarlyExit for Result<T, E> {
    type Remainder = T;
    type Return = Result<!, E>;

    #[inline]
    fn from_remainder(c: T) -> Self {
        Ok(c)
    }

    #[inline]
    fn branch(self) -> ControlFlow<Self::Return, T> {
        match self {
            Ok(c) => ControlFlow::Continue(c),
            Err(e) => ControlFlow::Break(Err(e)),
        }
    }
}

impl<T, E, F: From<E>> ops::FromReturn<Result<!, E>> for Result<T, F> {
    fn from_return(x: Result<!, E>) -> Self {
        match x {
            Err(e) => Err(From::from(e)),
        }
    }
}
```

### `Option`

```rust
impl<T> ops::EarlyExit for Option<T> {
    type Remainder = T;
    type Return = Option<!>;

    #[inline]
    fn from_remainder(c: T) -> Self {
        Some(c)
    }

    #[inline]
    fn branch(self) -> ControlFlow<Self::Return, T> {
        match self {
            Some(c) => ControlFlow::Continue(c),
            None => ControlFlow::Break(None),
        }
    }
}

impl<T> ops::FromReturn for Option<T> {
    fn from_return(x: <Self as ops::EarlyExit>::Return) -> Self {
        match x {
            None => None,
        }
    }
}
```

### `Poll`

These reuse `Result`'s residual type, and thus interconversion between `Poll` and `Result` is allowed without needing additional `FromReturn` implementations on `Result`.

```rust
impl<T, E> ops::EarlyExit for Poll<Result<T, E>> {
    type Remainder = Poll<T>;
    type Return = <Result<T, E> as ops::EarlyExit>::Return;

    fn from_remainder(c: Self::Remainder) -> Self {
        c.map(Ok)
    }

    fn branch(self) -> ControlFlow<Self::Return, Self::Remainder> {
        match self {
            Poll::Ready(Ok(x)) => ControlFlow::Continue(Poll::Ready(x)),
            Poll::Ready(Err(e)) => ControlFlow::Break(Err(e)),
            Poll::Pending => ControlFlow::Continue(Poll::Pending),
        }
    }
}

impl<T, E, F: From<E>> ops::FromReturn<Result<!, E>> for Poll<Result<T, F>> {
    fn from_return(x: Result<!, E>) -> Self {
        match x {
            Err(e) => Poll::Ready(Err(From::from(e))),
        }
    }
}
```

```rust
impl<T, E> ops::EarlyExit for Poll<Option<Result<T, E>>> {
    type Remainder = Poll<Option<T>>;
    type Return = <Result<T, E> as ops::EarlyExit>::Return;

    fn from_remainder(c: Self::Remainder) -> Self {
        c.map(|x| x.map(Ok))
    }

    fn branch(self) -> ControlFlow<Self::Return, Self::Remainder> {
        match self {
            Poll::Ready(Some(Ok(x))) => ControlFlow::Continue(Poll::Ready(Some(x))),
            Poll::Ready(Some(Err(e))) => ControlFlow::Break(Err(e)),
            Poll::Ready(None) => ControlFlow::Continue(Poll::Ready(None)),
            Poll::Pending => ControlFlow::Continue(Poll::Pending),
        }
    }
}

impl<T, E, F: From<E>> ops::FromReturn<Result<!, E>> for Poll<Option<Result<T, F>>> {
    fn from_return(x: Result<!, E>) -> Self {
        match x {
            Err(e) => Poll::Ready(Some(Err(From::from(e)))),
        }
    }
}
```

### `ControlFlow`

```rust
impl<B, C> ops::EarlyExit for ControlFlow<B, C> {
    type Remainder = C;
    type Return = ControlFlow<B, !>;

    fn from_remainder(c: C) -> Self {
        ControlFlow::Continue(c)
    }

    fn branch(self) -> ControlFlow<Self::Return, C> {
        match self {
            ControlFlow::Continue(c) => ControlFlow::Continue(c),
            ControlFlow::Break(b) => ControlFlow::Break(ControlFlow::Break(b)),
        }
    }
}

impl<B, C> ops::FromReturn for ControlFlow<B, C> {
    fn from_return(x: <Self as ops::EarlyExit>::Return) -> Self {
        match x {
            ControlFlow::Break(r) => ControlFlow::Break(r),
        }
    }
}
```

## Making the accidental `Option` interconversion continue to work

This is done with an extra implementation:
```rust
mod sadness {
    use super::*;

    /// This is a remnant of the old `NoneError` which is never going to be stabilized.
    /// It's here as a snapshot of an oversight that allowed this to work in the past,
    /// so we're stuck supporting it even though we'd really rather not.
    #[derive(Clone, Copy, PartialEq, PartialOrd, Eq, Ord, Debug, Hash)]
    pub struct PleaseCallTheOkOrMethodToUseQuestionMarkOnOptionsInAMethodThatReturnsResult;

    impl<T, E> ops::FromReturn<Option<!>> for Result<T, E>
    where
        E: From<PleaseCallTheOkOrMethodToUseQuestionMarkOnOptionsInAMethodThatReturnsResult>,
    {
        fn from_return(x: Option<!>) -> Self {
            match x {
                None => Err(From::from(
                    PleaseCallTheOkOrMethodToUseQuestionMarkOnOptionsInAMethodThatReturnsResult,
                )),
            }
        }
    }
}
```

## Use in `Iterator`

The provided implementation of `try_fold` is already just using `?` and `try{}`, so doesn't change.  The only difference is the name of the associated type in the bound:
```rust
fn try_fold<B, F, R>(&mut self, init: B, mut f: F) -> R
where
    Self: Sized,
    F: FnMut(B, Self::Item) -> R,
    R: EarlyExit<Remainder = B>,
{
    let mut accum = init;
    while let Some(x) = self.next() {
        accum = f(accum, x)?;
    }
    try { accum }
}
```

<!--
This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.
-->

# Drawbacks
[drawbacks]: #drawbacks

- While this handles a known accidental stabilization, it's possible that there's something else unknown that will keep this from being doable while meeting Rust's stringent stability guarantees.
- The extra complexity of this approach, compared to either of the alternatives considered the last time around, might not be worth it.
- This is the fourth attempt at a design in this space, so it might not be the right one either.
- As with all overloadable operators, users might implement this to do something weird.
- In situations where extensive interconversion is desired, this requires more implementations.
- Moving `From::from` from the desugaring to the implementations means that implementations which do want it are more complicated.

<!--
Why should we *not* do this?
-->

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Why `ControlFlow` pulls its weight

The previous RFC discussed having such a type, but ended up deciding that defining a new type for the desugar wasn't worth it, and just used `Result`.

This RFC does use a new type because one already [exists in nightly](https://doc.rust-lang.org/nightly/std/ops/enum.ControlFlow.html) under [the `control_flow_enum` feature gate](https://github.com/rust-lang/rust/issues/75744).
It's being used in [the library](https://github.com/rust-lang/rust/blob/fd34606ddf02d1e9364e459b373a6ad665c3d8a4/library/core/src/iter/traits/iterator.rs#L2239-L2252) and [the compiler](https://github.com/rust-lang/rust/blob/c609b2eaf323186a1167ec1a9ffa69a7d4a5b1b9/compiler/rustc_middle/src/ty/fold.rs#L184-L206), demonstrating that it's useful beyond just this desugaring, so the desugar might as well use it too for extra clarity.
There are also [ecosystem changes waiting on something like it](https://github.com/rust-itertools/itertools/issues/469#issuecomment-677729589), so it's not just a compiler-internal need.

## Methods on `ControlFlow`

On nightly there are a [variety of methods](https://doc.rust-lang.org/nightly/std/ops/enum.ControlFlow.html#implementations) available on `ControlFlow`.  However, none of them are needed for the stabilization of the traits, so they left out of this RFC.  They can be considered by libs at a later point.

There's a basic set of simple ones that could be included if desired, though:
```rust
impl<B, C> ControlFlow<B, C> {
	fn is_break(&self) -> bool;
	fn is_continue(&self) -> bool;
	fn break_value(self) -> Option<B>;
	fn continue_value(self) -> Option<C>;
}
```

## Traits for `ControlFlow`

`ControlFlow` derives a variety of traits where they have obvious behaviour.  It does not, however, derive `PartialOrd`/`Ord`.  They're left out as it's unclear which order, if any, makes sense between the variants.

For `Option`s, `None < Some(_)`, but for `Result`s, `Ok(_) < Err(_)`.  So there's no definition for `ControlFlow` that's consistent with the isomorphism to both types.

Leaving it out also leaves us free to change the ordering of the variants in the definition in case doing so can allow us to optimize the `?` operator.  (For a similar previous experiment, see [PR #49499](https://github.com/rust-lang/rust/pull/49499).)

## Naming the variants on `ControlFlow`

The variants are given those names as they serve the same purpose as the corresponding keywords when used in `Iterator::try_fold` or `Iterator::try_for_each`.

<!-- https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=f10bc2eab9db91273601c9e806989f7e -->
For example, this (admittedly contrived) loop
```rust
let mut sum = 0;
for x in iter {
    if x % 2 == 0 { continue }
    sum += x;
    if sum > 100 { break }
    continue
}
```
can be written as
```rust
let mut sum = 0;
iter.try_for_each(|x| {
    if x % 2 == 0 { return ControlFlow::Continue(()) }
    sum += x;
    if sum > 100 { return ControlFlow::Break(()) }
    ControlFlow::Continue(())
});
```
(Of course, one wouldn't normally use the `continue` keyword at the end of a `for` loop like that, but I've included it here to emphasize that even the `ControlFlow::Continue(())` as the final expression of the block it ends up working like the keyword would.)

## Why `ControlFlow` has `C = ()`

The type that eventually became `ControlFlow` was originally added way back in 2017 as [the internal-only type `LoopState`](https://github.com/rust-lang/rust/commit/b32267f2c1344d37c4aa30eccd5a9ab77642b3e6#diff-6f95fa6b66f447d11bb7507f832027237ee240310c159c74495a2363c82e76d7R357-R376) used to make some default implementations in `Iterator` easier to read.  It had no type parameter defaults.

[Issue #75744](https://github.com/rust-lang/rust/issues/75744) in 2020 started the process of exposing it, coming out of the [observation](https://github.com/rust-itertools/itertools/issues/469) that `Iterator::try_fold` isn't a great replacement for the deprecated-at-the-time `Itertools::fold_while` since using `Err` for a conceptual success makes code hard to read.

The compiler actually had [its own version of the type](https://github.com/rust-lang/rust/blob/515c9fa505e18a65d7f61bc3e9eb833b79a68618/src/librustc_data_structures/graph/iterate/mod.rs#L91-L94) in `librustc_data_structures` at the time:
```rust
pub enum ControlFlow<T> {
    Break(T),
    Continue,
}
```

The compiler was moved over to the newly-exposed type, and that inspired the creation of [MCP#374](https://github.com/rust-lang/compiler-team/issues/374), TypeVisitor: use ops::ControlFlow instead of bool.  Experience from that lead to flipping the type arguments in [PR#76614](https://github.com/rust-lang/rust/pull/76614) -- which also helped the original use cases in `Iterator`, where things like default implementation of `find` also want `C = ()`.  And these were so successful that it lead to [MCP#383](https://github.com/rust-lang/compiler-team/issues/383), TypeVisitor: do not hard-code a `ControlFlow<()>` result, having the visitors use `ControlFlow<Self::BreakTy>`.

As an additional anecdote that `C = ()` is particularly common, [Hytak mentioned the following](https://discord.com/channels/530598289813536771/530603542138847242/807920021728264193) on Discord in response to seeing a draft of this RFC:

> i didn't read your proposal in depth, but this reminds me of a recursive search function i experimented with a few days ago. It used a Result type as output, where Err(value) meant that it found the value and Ok(()) meant that it didn't find the value. That way i could use the `?` to exit early

So when thinking about `ControlFlow`, it's often best to think of it not like `Result`, but like an `Option` which short-circuits the other variant.  While it *can* flow a `Continue` value, that seems to be a fairly uncommon use in practice.

## Was this considered last time?

Interestingly, a [previous version](https://github.com/rust-lang/rfcs/blob/f89568b1fe5db4d01c4668e0d334d4a5abb023d8/text/0000-try-trait.md#using-an-associated-type-for-the-success-value) of RFC #1859 _did_ actually mention a two-trait solution, splitting the "associated type for ok" and "generic type for error" like is done here.  It's no longer  mentioned in the version that was merged.  To speculate, it may have been unpopular due to a thought that an extra traits just for the associated type wasn't worth it.

Current desires for the solution, however, have more requirements than were included in the RFC at the time of that version.  Notably, the stabilized `Iterator::try_fold` method depends on being able to create a `EarlyExit` type from the accumulator.  Including such a constructor on the trait with the associated type helps that separate trait provide value.

Also, ok-wrapping was decided [in #70941](https://github.com/rust-lang/rust/issues/70941), which needs such a constructor, making this ["much more appealing"](https://github.com/rust-lang/rust/issues/42327#issuecomment-379882998).

## Why not make the output a generic type?

It's helpful that type information can flow both ways through `?`.

- In the forward direction, not needing a contextual type means that `println!("{}", x?)` works instead of needing a type annotation.  (It's also just less confusing to have `?` on the same type always produce the same type.)
- In the reverse direction, it allows things like `let x: i32 = s.parse()?;` to infer the requested type from that annotation, rather than requiring it be specified again.

Similar scenarios exist for `try`, though of course they're not yet stable:

- `let y: anyhow::Result<_> = try { x };` doesn't need to repeat the type of `x`.
- `let x: i16 = { 4 };` works for infallible code, so for consistency it's good for `let x: anyhow::Result<i16> = try { 4 };` to also work (rather than default the literal to `i32` and fail).

## Trait and associated type naming

Bikeshed away!

## Why a "residual" type is better than an "error" type

Most importantly, for any type generic in its "output type" it's easy to produce a residual type using an uninhabited type.  That works for `Option` -- no `NoneError` residual type needed -- as well as for the `StrandFail<T>` type from the experience report.  And thanks to enum layout optimizations, there's no space overhead to doing this: `Option<!>` is a ZST, and `Result<!, E>` is no larger than `E` itself.  So most of the time one will not need to define anything additional.

In those cases where a separate type *is* needed, it's still easier to make a residual type because they're transient and thus can be opaque: there's no point at which a user is expected to *do* anything with a residual type other than convert it back into a known `EarlyExit` type.  This is different from the previous design, where less-restrictive interconversion meant that anything could be exposed via a `Result`.  That has lead to requests, [such as for `NoneError` to implement `Error`](https://github.com/rust-lang/rust/issues/46871#issuecomment-618186642), that are perfectly understandable given that the instances are exposed in `Result`s.  As residual types aren't ever exposed like that, it would be fine for them to implement nothing but `FromReturn` (and probably `Debug`), making them cheap to define and maintain.

## Use of `!`

This RFC uses `!` to be concise.  It would work fine with `convert::Infallible` instead if `!` has not yet stabilized, though a few more match arms would be needed in the implementations.  (For example, `Option::from_return` would need `Some(c) => match c {}`.)

## Moving away from the `Option`→`Result` interconversion

We could consider doing an edition switch to make this no longer allowed.

For example, we could have a different, never-stable `EarlyExit`-like trait used in old editions for the `?` desugaring.  It could then have a blanket impl, plus the extra interconversion one.

It's unclear that that's worth the effort, however, so this RFC is currently written to continue to support it going forward.  Notably, removing it isn't enough to solve the annotation requirements, so the opportunity cost feels low.

## Why `FromReturn` is the supertrait

It's nicer for `try_fold` implementations to just mention the simpler `EarlyExit` name.  It being the subtrait means that code needing only the basic scenario can just bound on `EarlyExit` and know that both `from_remainder` and `from_return` are available.

## Default `Return` on `FromReturn`

The default here is provided to make the basic case simple.  It means that when implementing the trait, the simple case (like in `Option`) doesn't need to think about it -- similar to how you can `impl Add for Foo` for the homogeneous case even though that trait also has a generic parameter.

## `FromReturn::from_return` vs `Return::into_try`

Either of these directions could be made to work.  Indeed, an early experiment while drafting this had a method on a required trait for the residual that created the type implementing `EarlyExit` (not just the associated type).  However that method was removed as unnecessary once `from_return` was added, and then the whole trait was moved to future work in order to descope the RFC, as it proved unnecessary for the essential `?`/`try_fold` functionality.

A major advantage of the `FromReturn::from_return` direction is that it's more flexible with coherence when it comes to allowing other things to be converted into a new type being defined.  That does come at the cost of higher restriction on allowing the new type to be converted into other things, but reusing a residual can also be used for that scenario.

Converting a known residual into a generic `EarlyExit` type seems impossible (unless it's uninhabited), but consuming arbitrary residuals could work -- imagine something like
```rust
impl<R: std::fmt::Debug> FromReturn<R> for LogAndIgnoreErrors {
    fn from_return(h: H) -> Self {
        dbg!(h);
        Self
    }
}
```
(Not that that's necessarily a good idea -- it's plausibly *too* generic.  This RFC definitely isn't proposing it for the standard library.)

And, ignoring the coherence implications, a major difference between the two sides is that the target type is typically typed out visibly (in a return type) whereas the source type (going into the `?`) is often the result of some called function.  So it's preferable for any behaviour extensions to be on the type that can more easily be seen in the code.

<!--
- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?
-->

# Prior art
[prior-art]: #prior-art

Previous approaches used on nightly
- The original [`Carrier` trait](https://doc.rust-lang.org/1.16.0/core/ops/trait.Carrier.html)
- The next design with a [`EarlyExit` trait](https://doc.rust-lang.org/1.32.0/core/ops/trait.EarlyExit.html) (different from the one here)

Thinking from the perspective of a [monad](https://doc.rust-lang.org/1.32.0/core/ops/trait.EarlyExit.html), `EarlyExit::from_remainder` is similar to `return`.

<!--
Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.
-->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Bikesheds: `EarlyExit`/`FromReturn`/`EarlyExit::Remainder`/`EarlyExit::Return` all might have better names.  This RFC has left it as `EarlyExit` mostly because that meant not touching all the `try_fold` implementations in the prototype.  I've long liked [parasyte's "bubble" suggestion](https://internals.rust-lang.org/t/bikeshed-a-consise-verb-for-the-operator/7289/29?u=scottmcm) as a name, but maybe sticking with the previous one is best.
- Structure: The three methods could be split up further

<!--
- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?
-->

# Future possibilities
[future-possibilities]: #future-possibilities

While it isn't directly used in this RFC, a particular residual type can be used to define a "family" of types which all share that residual.

For example, one could define a trait like this one:
```rust
pub trait GetCorrespondingEarlyExitType<EarlyExitRemainderType>: Sized {
    /// The type from the original type constructor that also has this residual type,
    /// but has the specified Remainder type.
    type EarlyExitType: EarlyExit<Remainder = EarlyExitRemainderType, Return = Self>;
}
```

With corresponding simple implementations like these:
```rust
impl<T> GetCorrespondingEarlyExitType<T> for Option<!> {
    type EarlyExitType = Option<T>;
}

impl<C, B> ops::GetCorrespondingEarlyExitType<C> for ControlFlow<B, !> {
    type EarlyExitType = ControlFlow<B, C>;
}
```

And thus allow code to put whatever value they want into the appropriate type from the same family.

This can be thought of as the type-level inverse of `EarlyExit`'s associated types: It splits them apart, and this puts them back together again.

(Why is this not written using Generic Associated Types (GATs)?  Because it allows implementations to work with only specific types, or with generic-but-bounded types.  Anything using it can bound to just the specific types needed for that method.)

A previous version of this RFC included a trait along these lines, but it wasn't needed for the stable-at-time-of-writing scenarios.  Furthermore, some experiments demonstrated that having a bound in `EarlyExit` requiring it (something like `where Self::Return: GetCorrespondingEarlyExitType<Self::Remainder>`) wasn't actually even helpful for unstable scenarios, so there was no need to include it in normative section of the RFC.

## Possibilities for `try_find`

Various library methods, such as `try_map` for arrays ([PR #79713](https://github.com/rust-lang/rust/pull/79713#issuecomment-739075171)), would like to be able to do HKT-like things to produce their result types.  For example, `Iterator::try_find` wants to be able to return a `Foo<Option<Item>>` from a predicate that returned a `Foo<bool>`.

That could be done with an implementation such as the following:
```rust
fn try_find<F, R>(
    &mut self,
    f: F,
) -> <R::Return as ops::GetCorrespondingEarlyExitType<Option<Self::Item>>>::EarlyExitType
where
    Self: Sized,
    F: FnMut(&Self::Item) -> R,
    R: ops::EarlyExit<Remainder = bool>,
    R::Return: ops::GetCorrespondingEarlyExitType<Option<Self::Item>>,
{
    #[inline]
    fn check<F, T, R>(mut f: F) -> impl FnMut((), T) -> ControlFlow<Result<T, R::Return>>
    where
        F: FnMut(&T) -> R,
        R: EarlyExit<Remainder = bool>,
    {
        move |(), x| match f(&x).branch() {
            ControlFlow::Continue(false) => ControlFlow::Continue(()),
            ControlFlow::Continue(true) => ControlFlow::Break(Ok(x)),
            ControlFlow::Break(r) => ControlFlow::Break(Err(r)),
        }
    }

    match self.try_fold((), check(f)) {
        ControlFlow::Continue(()) => EarlyExit::from_remainder(None),
        ControlFlow::Break(Ok(x)) => EarlyExit::from_remainder(Some(x)),
        ControlFlow::Break(Err(r)) => <_>::from_return(r),
    }
}
```

Similarly, it could allow `EarlyExit` to automatically provide an appropriate `map` method:
```rust
fn map<T>(self, f: impl FnOnce(Self::Remainder) -> T) -> <Self::Return as GetCorrespondingEarlyExitType<T>>::EarlyExitType
where
    Self::Return: GetCorrespondingEarlyExitType<T>,
{
    match self.branch() {
        ControlFlow::Continue(c) => EarlyExit::from_remainder(f(c)),
        ControlFlow::Break(r) => FromReturn::from_return(r),
    }
}

```

## Possibilities for `try{}`

A core problem with [try blocks](https://doc.rust-lang.org/nightly/unstable-book/language-features/try-blocks.html) as implemented in nightly, is that they require their contextual type to be known.

That is, the following never compiles, no matter the types of `x` and `y`:
```rust
let _ = try {
	foo(x?);
	bar(y?);
	z
};
```

This usually isn't a problem on stable, as the `?` usually has a contextual type from its function, but can still happen there in closures.

But with something like `GetCorrespondingEarlyExitType`, an alternative desugaring becomes available which takes advantage of how the residual type preserves the "result-ness" (or whatever-ness) of the original value.  That might turn the block above into something like the following:
```rust
fn helper<C, R: GetCorrespondingEarlyExitType<C>>(r: R) -> <R as GetCorrespondingEarlyExitType<C>>::EarlyExitType
{
	FromReturn::from_return(h)
}

'block: {
	foo(match EarlyExit::branch(x) {
		ControlFlow::Continue(c) => c,
		ControlFlow::Break(r) => break 'block helper(r),
	});
	bar(match EarlyExit::branch(y) {
		ControlFlow::Continue(c) => c,
		ControlFlow::Break(r) => break 'block helper(r),
	});
	EarlyExit::from_remainder(z)
}
```
(It's untested whether the inference engine is smart enough to pick the appropriate `C` with just that -- the `Remainder` associated type is constrained to have a `Continue` type matching the generic parameter, and that `Continue` type needs to match that of `z`, so it's possible.  But hopefully this communicates the idea, even if an actual implementation might need to more specifically introduce type variables or something.)

That way it could compile so long as the `EarlyExitType`s of the residuals matched.  For example, [these uses in rustc](https://github.com/rust-lang/rust/blob/7cf205610e1310897f43b35713a42459e8b40c64/compiler/rustc_codegen_ssa/src/back/linker.rs#L529-L573) would work without the extra annotation.

Now, of course that wouldn't cover anything.  It wouldn't work with anything needing error conversion, for example, but annotation is also unavoidable in those cases -- there's no reasonable way for the compiler to pick "the" type into which all the errors are convertible.

So a future RFC could define a way (syntax, code inspection, heuristics, who knows) to pick which of the desugarings would be best.  (As a strawman, one could say that `try { ... }` uses the "same family" desugaring whereas `try as anyhow::Result<_> { ... }` uses the contextual desugaring.)  This RFC declines to debate those possibilities, however.

*Note that the `?` desugaring in nightly is already different depending whether it's inside a `try {}` (since it needs to block-break instead of `return`), so making it slightly more different shouldn't have excessive implementation cost.*

## Possibilities for `yeet`

As previously mentioned, this RFC neither defines nor proposes a `yeet` operator.  However, like the previous design could support one with its `EarlyExit::from_error`, it's important that this design would be sufficient to support it.

*`yeet` is a [bikeshed-avoidance](https://twitter.com/josh_triplett/status/1248658754976927750) name for `throw`/`fail`/`raise`/etc, used because it definitely won't be the final keyword.*

Because this "residual" design carries along the "result-ness" or "option-ness" or similar, it means there are two possibilities for a desugaring.

- It could directly take the residual type, so `yeet e` would desugar directly to `FromReturn::from_return(e)`.
- It could put the argument into a special residual type, so `yeet e` would desugar to something like `FromReturn::from_return(Yeeted(e))`.

These have various implications -- like `yeet None`/`yeet`, `yeet Err(ErrorKind::NotFound)`/`yeet ErrorKind::NotFound.into()`, etc -- but thankfully this RFC doesn't need to discuss those.  (And please don't do so in the GitHub comments either, to keep things focused, though feel free to start an IRLO or Zulip thread if you're so inspired.)

<!--
Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. EarlyExit to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
-->
