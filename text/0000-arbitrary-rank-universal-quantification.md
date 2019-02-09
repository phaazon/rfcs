- Feature Name: arbitrary_rank_universal_quantification
- Start Date: 2019-02-09
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

> Disclaimer: this RFC is currently in a Pre-RFC state. It is meant to be shared and widely
> discussed on Reddit (/r/rust). Please comment and provide feedback so that the RFC can be
> enhanced.
>
> Discussion happens [here](https://www.reddit.com/r/rust/comments/aoxk14/prerfc_arbitraryrank_universal_quantification).

This RFC brings *arbitrary-rank explicit universal quantification* to Rust (also known as the more
popular [rank-N types]). Such quantification allows to create universally quantified functions at
any ranks (instead of the first, *rank 1* current situation). Hence, the following is now legal and
correct syntax:

```rust
// a rank-2 function
fn foo2<F>(f: F) where F: for<T> Fn(T) -> String {
  println!("called with a string: {}", f("hello"));
  println!("called with an u32: {}", f(42_u32));
  println!("called with unit: {}", f(()));
}

// a rank-3 function, less common
fn foo3<F>(f: F) where F: for<G: for<T> Fn(T) -> u32> Fn(G) -> String {
  println!("with len: {}", f(String::len));
  println!("with a constant value: {}", f(|_| 42));
}

// we can use the where-clause as well
fn foo3<F>(f: F) where F: for<G> Fn(G) -> String where G: for<T> Fn(String) -> u32;

// rank-4 with where-clause:
fn foo4<F>(f: F) where F: for<G> Fn(G) -> String where G: for<H> Fn(H) -> u32 where H: for<T> Fn(T);
```

# Motivation
[motivation]: #motivation

A typical motivation for arbitrary-rank explicit quantification is to strengthen the higher-order
functions experience. Consider the following code:

```rust
trait HasSlices {
  fn visit_slice<F>(&self, f: F) where F: FnMut(&[_]);
}
```

That trait represents all types that have slices available in any possible way (or generated on the
fly) and that can be visited. You could for instance define such a type like this:

```rust
struct Gorgonzola {
  octopus: [String; 12],
  wales: Vec<u32>
}
```

However, the current implementation of `HasSlices` lacks a type in the `where` clause: what should
we put for the `_`?

## First idea: rank-1 universal quantification

Currently, Rust support explicit rank-1 universal quantification. What it means is that you can
only specify a single level of universal quantification. We end up with this trait definition:

```rust
trait HasSlices {
  fn visit_slice<F, T>(&self, f: F) where F: FnMut(&[T]);
}
```

This kind of universal quantification gives you a contract that says that you will be able to
traverse a `Foo: HasSlices` with a universally quantified function. But its argument arge also
universally quantified **at the same time**, with the same quantifier. What it means is that when
you choose `F`, you also have to choose `T` for a call to `visit_slice` to be possible. Because of
the method being monomorphized, if you want to call it in a recursive way, for instance, you will
be stuck to both `F` and `T`. Not very useful in our case. If we try to implement that trait
for `Gorgonzola`:

```rust
impl HasSlices for Gorgonzola {
  fn visit_slice<F, T>(&self, f: F) where F: FnMut(&[T]) {
    // here, T is a free type variable, so we cannot really use F!
    // we could use it if we had a &[T], but we don’t here
  }
}
```

Instead, what we want is rank-2 universal quantification. We want the `visit_slice` method to be
parameterized by `F` only, and we want that `F` to be parameterized by `T` — which is exactly what
rank-2 universal quantification is all about. Currently, Rust doesn’t support that as a first-class
concept in the language. We have to do it all by ourselves.

```rust
trait HasSlices {
  fn visit_slice<F>(&self, f: F) where F: SliceVisitor;
}

trait SliceVisitor {
  fn visit<T>(&mut self, slice: &[T]);
}
```

Here, `HasSlices::visit_slice` is parameterized by `F`, which must implement the `SliceVisitor`.
`SliceVisitor::visit` is parameterized by `T`, so we have manually crafted a rank-2 universally
quantified function. We can implement it this way:

```rust
impl HasSlices for Gorgonzola {
  fn visit_slice<F>(&self, f: F) where F: SliceVisitor {
    f.visit(&self.octopus);
    f.visit(&self.wales[..]);
  }
}
```

We can then implement a specific visitor to do anything with the slices:

```rust
struct PrintVisitor;

impl SliceVisitor for PrintVisitor {
  fn visit<T>(&mut self, slice: &[T]) {
    println!("visiting a slice of length {}", slice.len());
  }
}

struct CountSlicesVisitor {
  count: usize
}

impl SliceVisitor for PrintVisitor {
  fn visit<T>(&mut self, _: &[T]) {
    self.count += 1;
  }
}
```

This RFC provides a new, non-breaking change notation to express rank-N universal quantification,
removing the need of a trait:

```rust
trait HasSlices {
  fn visit_slice<F>(&self, f: F) where F: for<T> FnMut(&[T]);
}

impl HasSlices for Gorgonzola {
  fn visit_slice<F>(&self, f: F) where F: FnMut<T>(&[T]) {
    f(&self.octopus);
    f(self.wales.slice());
  }
}
```

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Rank-N universal quantification

Polymorphic functions have universally quantified type variables, expressed via the `<…>`
notation:

```rust
fn rank1<A, B, C>(a: A, b: &B, c: Option<C>);
```

Such functions use *rank-1 universal quantification*, meaning that all variables have the same
universal rank and are **all set at once by the caller, at the same time**. It is possible to use
*rank-2 universal quantification*, meaning that type variables won’t be quantified all at the same
time.

```rust
fn rank2<A, B>(a: A, b: &B) where A: for<T> Fn(T) -> String where T: Display;
```

Here, `rank2` will have both `A` and `B` type set by the caller, but the type of `T` is set by the
function’s body directly. That enables to pass a rank-1 function as `A` that will work for any type
used by the implementation of the `rank2` function. With a rank-1 function, that is not possible,
unless you use a trait to manually craft a rank-2 function:

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

  - Existential quantification is enabled with the `for<…>` language construct.
  - One `for<…>` increments the current rank by one. All generic functions are by default *rank-1*.
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

## Turning manually-crafted universal quantification to arbitrary universal quantification

It is possible that you already have some code that uses the trait trick explained above with
`GetString`. Migrating to rank-N universal quantification is not that hard: remove the trait and
replace it with a `for<…>`. If you had trait bound on the type variable on your trait method, you
can move it inside the `for<…>` construct.

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
is.

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
might confuse people by making types look more convoluted. Arbitrary universal quantification is not
an easy concept and should be considered as advanced Rust teaching. However, since it’s a very
powerful concept (largely used in the Haskell community, for instance), more crates could migrate to
it and expose it in their public APIs. That might drive newcomers overwhelmed.

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
    - However, those traits must be given in a name, which creates other issues (naming things)
      while we could ditch the names for a more direct, first-class language construct.
    - Also, since the trait must be public, it can be used in other places than where the function
      uses it. This might create confusion as the trait can now live on its own.

# Prior art
[prior-art]: #prior-art

Haskell has had [rank-N types] for a while now. In Haskell, this is introduced with the `forall`
keyword and where it is placed.

```haskell
{-# LANGUAGE RankNTypes #-}

foo :: (forall a. (Show a) => a -> String) -> String
foo f = f "Hello " <> f 21

foo show -- = "Hello 21"
```

# Unresolved questions
[unresolved]: #unresolved-questions

N/A, currently.

[rank-N types]: https://wiki.haskell.org/Rank-N_types
[HRTBs — Higher-Rank Trait Bounds]: https://doc.rust-lang.org/nomicon/hrtb.html
