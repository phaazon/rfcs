- Feature Name: rank_n_hrtb
- Start Date: 2019-02-09
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

> Disclaimer: this RFC is currently in a pre-RFC state. It is meant to be shared and widely
> discussed on Reddit (/r/rust). Please comment and provide feedback so that the RFC can be
> enhanced.
>
> - Last version of the pre-RFC [here]().
> - Previous version of the pre-RFC [here](https://github.com/phaazon/rfcs/blob/rfc/arbitrary-rank-universal-quantification/text/0000-arbitrary-rank-universal-quantification.md).
> - Previous pre-RFC proposal discussion [here](https://www.reddit.com/r/rust/comments/aoxk14/prerfc_arbitraryrank_universal_quantification).

Rust has this powerful feature called [HRTBs](https://doc.rust-lang.org/beta/nomicon/hrtb.html),
which stands for _Higher-Rank Trait Bounds_. HRTBs allow one to _universally quantify_ a
_trait bound_ (typically, `Fn`, `FnOnce` and `FnMut` but not limited to) over one or several
lifetimes via the language construct `for<'a, 'b, …>`. That feature allows `T: for<'a> …` to be
used with _any_ lifetimes. The `'a` lifetime only appears in the trait bound, not in the function
introducing the `T` type. That can also be read as _`T` has a trait that works `for all` lifetime_.

This RFC extends the `for<>` language construct to:

- Introduce type variables — e.g. `for<T>`.
- Nest HRTBs to introduce [_rank-N quantification_] — e.g. `for<F: for<T> Fn(T) -> String>`.

# Motivation
[motivation]: #motivation

The motivation for this RFC requires some already advanced Rust code as it’s not an immediate
intuitive feature. Consider the following problem:

- We want to write a function that forward its arguments to functions that we want to _time_.
- In order to time those, we want our function to take another function that will implement the
  timing mechanism.

```rust
fn process<F>(x: u32, timer: F) {
  // time this
  foo(x);

  // time this too
  bar(x);
}
```

## Not-so-intuitive solution

In order to time the functions, we will want to wrap them in closures. However, closures are
unnamed, so we cannot refer to their types _explicitly_.

```rust
fn process<F>(x: u32, timer: F) {
  timer("foo", || foo(x));
  timer("bar", || bar(x));
}
```

But now, what’s the type of `timer`?

```rust
fn process<F, G>(x: u32, timer: F)
where
  F: Fn(G),
  G: Fn(u32)
{
  timer("foo", || foo(x));
  timer("bar", || bar(x));
}
```

This doesn’t work, because both `F` and `G` must be resolved at the same time, which means that:

- `F` is picked by the caller of `process`.
- `G` is picked by the caller of `process`.
- `G` has a single type, defined by the caller, so we cannot use any of our closures here.

The current solution to that problem in Rust is to recognize that we want to _universally quantify_
`F` over _functions taking `u32`_. In order to do that, unfortunately, we have no other choices but
to use a trait:

```rust
trait Timer {
  fn time<F>(&self, fn_name: &str, f: F) where F: Fn();
}

struct DebugTimer;

impl Timer for DebugTimer {
  fn time<F>(&self, fn_name: &str, f: F) where F: Fn() {
    println!("start recording {}", fn_name);
    f();
    println!("end recording {}", fn_name);
  }
}
```

With this trait, we can then implement our `process` function:

```rust
fn process<F>(x: u32, timer: F) where F: Timer {
  timer("foo", || foo(x));
  timer("bar", || bar(x));
}
```

However, this is not very intuitive nor convenient:

- Users have to declare a type and implement a trait to perform such a trivial task.
- We have to find a name to a trait that makes sense for a single function, which may
  sound a bit overkill.

## The suggested solution

What we really intended to do here was to state that `F` works _for all types which implements the
`Fn()` trait_. That statement is visible in the `time` method of the `Timer` trait: it is
universally quantified over the `F` type variable which is defined as `F: Fn()`. What we’ve been
doing is hiding a universally quantified call behind a trait bound. We could removing that hiding
so that the trait is not needed anymore and the syntax feels more natural.

Because Rust already has a syntax to express _for all_ ideas, we augment it to support types and it
now becomes the following:

```rust
fn process<F>(x: u32, timer: F) where F: for<'a, G> Fn(&'a str, G) where G: Fn() {
  timer("foo", || foo(x));
  timer("bar", || bar(x));
}
```

Similar syntax:

```rust
fn process<F>(x: u32, timer: F) where F: for<'a, G: Fn()> Fn(&'a str, G) {
  timer("foo", || foo(x));
  timer("bar", || bar(x));
}
```

Even more condensed:

```rust
fn process<F: for<'a, G: Fn()> Fn(&'a str, G)>(x: u32, timer: F) {
  timer("foo", || foo(x));
  timer("bar", || bar(x));
}
```

This kind of quantification is referred to as _rank-2_, because we have two ranks (read levels) of
deepening of type variables:

1. First level at the function level: it’s `F`.
2. Second level at the trait level of `F`: it’s `G`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Rank-N HRTBs (rank-N universal quantification)

Polymorphic functions have universally quantified type variables, expressed via the `<…>`
notation:

```rust
fn rank1<A, B, C>(a: A, b: &B, c: Option<C>);
```

Such functions use *rank-1 universal quantification*, meaning that all variables have the same
universal rank and are **all set at once by the caller, at the same time**. It is not possible to
substitute a type variable inside the callee. In order to do that, it is possible to use *rank-2
universal quantification*, meaning that type variables won’t be quantified all at the same
time.

```rust
fn rank2<A, B>(a: A, b: &B) where A: for<T> Fn(T) -> String where T: Display;
```

Here, `rank2` will have both `A` and `B` types set by the caller, at the same level, but the type
of `T` is set by the function’s body directly. That enables to pass a function that will work for
any type used by the implementation of the `rank2` function. With a rank-1 function, that is not
possible, unless you use a trait to manually craft a rank-2 function:

```rust
trait GetString {
  fn get_string<T>(&self) -> String where T: Display;
}

fn rank2<A, B>(a: A, b: &B) where A: GetString;
```

This feature is called *rank-N universal quantification*, which means that you can use it for any
kind of nesting:

```rust
fn rank3<A>(a: A) where A: for<F> Fn(F) -> String where F: for<T> Fn(&Option<T>) -> u32;
```

The syntax works as follows:

  - Universal quantification is enabled with the `for<…>` language construct.
  - One `for<…>` increments the current rank by one. All generic functions are by default *rank-1*.
    A function rank is determined by its deepest `for` rank.
  - Inside a `for<…>` construct, the syntax is the same as the list of type variables and lifetimes
    you would find in *rank-1* functions, such as in `fn foo<…>();`.
  - Especially, trait bounds can be inlined — e.g. `for<T: Clone>` — or in a `where`-clause.
  - As with typical and regular function declarations, the `where` clause is found after the trait
    bound — e.g. `for<T> Fn(T) -> usize where T: Clone`.
  - Rank-N universal quantification is enabled by nesting `for<…>` inside each other.
    - This is very explicit when using inlined trait bounds — e.g. `for<F: for<T> Fn(T) -> usize>`,
      used for rank-3 universal quantification.
    - The nesting is less visible with `where` clauses but still there:
      `for<F> _ where F: for<T> Fn(T) -> usize`.
  - Rank-N universal quantification can be used anywhere trait bounds can appear.
  - Ambiguities with `where` clauses can be resolved by wrapping trait bounds in parenthesis `( )`:
    - `F: for<A, B> Fn(A) -> B where A: (for<C, D> Fn() -> C where C: Debug, D: Debug), B: Clone`.
    - `F: for<A, B> Fn(A) -> B where B: Clone, (A: for<C, D> Fn() -> C where C: Debug, D: Debug)`.
    - Parenthesis are mandatory as soon as we are deeper than rank-2.
  - _Shadowing_ type variables is forbidden as it would result in highly complex and readable
    types. E.g., the folowing is forbidden:
    - `F: for<F> Fn(F) -> String where F: FnOnce(u32) -> String`.

## Turning manually-crafted universal quantification to arbitrary universal quantification

It is possible that you already have some code that uses the trait trick explained above with
`GetString`. Migrating to rank-N HRTBs is not that hard: remove the trait and replace it with a
`for<…>`. If you had trait bounds on the type variables on your trait method, you can move them
inside the `for<…>` construct.

## Error messages

When a type variable appears in a trait bound, if it’s universally quantified, you will get an
error:

```rust
fn errored<F>(f: F) where F: Fn() -> T
```

generates the following error:

```
error[E0412]: cannot find type `T` in this scope
 --> src/main.rs:4:34
  |
4 | fn foo<F>(f: F) where F: Fn() -> T {
  |                                  ^ did you mean a universally quantified type parameter?
  = note: cannot find type `T`
          introduce universal quantification with `for<T>`

  |
4 | fn foo<F>(f: F) where F: Fn() -> T {
  |                          ^ for<T>
```

## How to teach that to new Rust programmers

For newcomers, generic functions are mandatory in order to understand what universal quantification
is. The current introduction to HRTBs from the nomicon is needed too (actually, that should be in
the book).

The concept of _rank-N HRTBs_ is just a natural extension to HRTBs and should be read as _for all_
types. The _rank_ of a type variable defines its depth in the quantification tree, which
authorizes, given a function `foo`.

- Rank-1 type variables are picked by the caller of `foo`.
- Rank-2 type variables are picked by the `foo` implementation. If it’s still too hard to
  visualize, try imagining that the implementation of `foo` is the _new global scope_ and that the
  type variables are just polymorphic variables you can set to whatever you want; like if you were
  using a polymorphic function, you can decide to use it with different types if you want. You end up
  in a somewhat rank-1 case again, if that ever helps.
- Rank-3 type variables are akin to rank-2 type variables but with one level deeper.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This feature interacts with [HRTBs — Higher-Rank Trait Bounds]. HRTBs have already made the syntax
`for<…>` common and useful for lifetimes, introducing *rank-2* lifetimes. It is quite logical and
natural to authorize doing so with types as well.

Such a feature has a current workaround (i.e. universal quantification via a trait). Each new trait
adds a new rank: *rank-3* requires two additional traits.

# Drawbacks
[drawbacks]: #drawbacks

The syntax allows for a lot of expressivity power that will remove traits in favor of a much more
concise and natural, already existing syntax (i.e. HRTBs). This can be seen as a drawback as it
might confuse people by making types look more convoluted. Rank-N universal quantification is not
an easy concept and should be considered as advanced Rust teaching. However, since it’s a very
powerful concept (largely used in the Haskell community, for instance), more crates could migrate to
it and expose it in their public APIs. That might drive newcomers overwhelmed.

Rank-3 universal quantification is _hard_ to wrap your finger around and is also rarely used. It has
its use cases but it’s very likely most of rustaceans will never encounter them. Maybe the current
document should consider rank-2 universal quantification first and only?

# Rationale and alternatives
[alternatives]: #alternatives

Traits are currently used to implement, in most of cases, *rank-2* universal quantification. The
current syntax, based on `for`, is a continuation of HRTBs, but others designs were considered:

  - Introduce new ranks by using the `<…>` directy on the trait — e.g.
    `fn foo<F>(f: F) where F: Fn<T>() -> T`.
    - This syntax feels natural as it places the type variables at the same place as with regular
      functions.
    - However, it would feel weird to have two different syntaxes for *rank-N* types and for
      *rank-N* lifetimes.
  - Do nothing and keep with traits. This is fine as it’s already doable and *rank-2* *“only”*
    requires an extra trait.
    - However, those traits must be given a name, which creates other issues (a trait for a single
      function, which is overkill; naming the trait; creating _dummy_ types to implement the trait;
      etc.) while we could ditch the names for a more direct, first-class language construct.
    - Also, since the trait must be public, it can be used in other places than where the function
      uses it. This might create confusion as the trait can now live on its own.

# Prior art
[prior-art]: #prior-art

Haskell has had [rank-N types] for a while now. In Haskell, this is introduced with the `forall`
keyword and where it is placed.

```haskell
{-# LANGUAGE RankNTypes #-}

-- rank 2
foo2 :: (forall a. (Show a) => a -> String) -> String
foo2 f = f "Hello " <> f 21

foo2 show -- = "Hello 21"
```

# Unresolved questions
[unresolved]: #unresolved-questions

## HRTBs and type language theory scares people

We need to re-work the _How we teach that_ section so that it provides a deeper understanding to
people who are not used to type language theory.

[_rank-N quantification_]: https://wiki.haskell.org/Rank-N_types
[HRTBs — Higher-Rank Trait Bounds]: https://doc.rust-lang.org/nomicon/hrtb.html
