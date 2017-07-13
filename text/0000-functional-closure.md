- Feature Name: functional_closure
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

A use of Rust's closure syntax creates a unique, unnamed type implementing the
special `Fn`, `FnMut`, or `FnOnce` traits. This RFC proposes extending that
behavior to an implementation of any "functional" trait - one that has a single
non-defaulted, non-static method. This is very similar to the behavior of Java
8's closures.

# Motivation
[motivation]: #motivation

There are contexts in which you want a specific trait rather than the more
generic `Fn*` traits. For example, the trait may provide defaulted methods in
addition to the "main" method. Take the `Iterator` trait. It has a single
required method, `next`, with a great deal of default methods added on top.
Imagine you have a `Foo::bar` method which returns an `Option<u32>`, and you
want to pass that into a function expecting an `Iterator<Item = u32>`. Currently
you have to manually manually create a wrapper struct:

```rust
struct FooIterator(Foo);

impl Iterator for FooIterator {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        self.0.bar()
    }
}

let foo = ...;
some_function(FooIterator(foo));
```

However, this could be simplified to just

```rust
let mut foo = ...;
some_function(|| foo.bar());
```

A somewhat common convention is to add a blanket implementation of a trait for
types implementing the relevant closure type. In the `Iterator` case, this would
hypthetically look like:

```rust
impl<T, R> Iterator for T
where
    T: FnMut() -> Option<R>
{
    type Item = R;

    fn next(&mut self) -> Option<R> {
        self()
    }
}
```

However, there are limitations to this approach. It has to be explicitly
implemented for each trait - this was not done for `Iterator`, for example.
It also breaks backwards compatibility to add after the fact since downstream
concrete implementations for types which also implement `FnMut() -> Option<R>`
will suddenly conflict.

In addition, blanket implementations like this don't compose well with *other*
implementations you also want. For example, `Iterator` is implemented for
mutable references to iterators:

```rust
impl<'a, T> Iterator for &'a mut T
where
    T: ?Sized + Iterator
{
    type Item = T::Item;

    fn next(&mut self) -> Option<T::Item> {
        (**self).next()
    }
}
```

This implementation conflicts with the proposed implementation for closures.

In other contexts, you have a trait with only one method, but common
implementations are complex enough that simply using a `Fn*` trait is not
appropriate. For example, consider the Iron
[`Handler`](https://docs.rs/iron/0.5.1/iron/middleware/trait.Handler.html)
trait. It is responsible for processing an HTTP request and producing the
response. It has a blanket implementation for all `Fn` implementations with a
matching signature, which allows for very convenient "toy" server
implementations:

```rust
fn main() {
    Iron::new(|_: &mut Request| {
        Ok(Response::with((status::Ok, "Hello World!")))
    }).http("localhost:3000").unwrap();
}
```

However, concrete types also implement `Handler`. Take for example the
[`Router`](https://docs.rs/router/0.5.1/router/struct.Router.html) type, which
selects a sub-`Handler` based on the request's endpoint.

There are also contexts where that workaround will not work at all. Consider,
for example, Serde's `DeserializeSeed` trait:

```rust
pub trait DeserializeSeed<'de>: Sized {
    type Value;

    fn deserialize<D>(self, deserializer: D) -> Result<Self::Value, D::Error>
    where
        D: Deserializer<'de>;
}
```

We can't play the same trick here - note that the `deserialize` method itself is
parameterized. We would need to require that `T` implements `FnOnce(D)` for
*all* `D: Deserializer`, this requires Higher Ranked Trait Bounds (HRTB) syntax
which isn't supported in Rust's type system (this is supported for lifetimes
with the `where T: for <'a> FnOnce(&'a D)` syntax).

# Detailed design
[design]: #detailed-design

A trait is *functional* if it:

* Has a single method without a default implementation,
* that method is not static,
* and has no supertraits which are not auto traits.

For example, `Iterator` and `DeserializeSeed<'de>` above are both functional.

Examples of traits which are not functional:

```rust
// Method is static.
pub trait Default {
    fn default() -> Self;
}

// Two methods which need implementation.
pub trait Bar {
    fn method_a(&self);

    fn method_b(&self);
}

// No method which needs an implementation.
pub trait Baz {
    fn defaulted(&self) {}
}

// Has a non-auto supertrait
pub trait Buz: Eq {
    fn buz(&self);
}
```

Note that, strangely, `Fn` and `FnMut` as they exist today are *not* functional
by this definition! `Fn` and `FnMut` have `FnMut` and `FnOnce` as supertraits
respectively. We will treat them as a special case. This is somewhat
unfortunate - we may also be able to adjust the definitions of the `Fn*` traits
and replace the supertrait bounds with blanket implementations. We have
some flexibility here because the definitions of the `Fn*` traits are not
stable.

If the type of a closure literal is inferred to be a type implementing a
functional trait, the compiler will generate an appropriate implementation of
that trait.

This implementation generation will also involve the selection of associated
types. This is already handled today, as the return type of `Fn*` traits is an
associated type. In theory, there may exist associated types which are not
involved in the signature of the functional method. In this case, those
associated types may need to be constrained externally:

```rust
trait Foo {
    // Can be inferred from the closure.
    type Bar;

    // Requires separate hinting.
    type Baz;

    fn foo(&self) -> Self::Bar;
}

fn use_a_foo<F>(foo: F)
where
    F: Foo<Baz = u32>
{
    // ...
}

// valid, as `Baz`'s type is selected by `use_a_foo`'s signature.
use_a_foo(|| 0i32);
```

Like integer type inference has an implicit fallback to `i32` if the exact type
can't otherwise be inferred, closure literals will have an implicit fallback to
the `Fn*` traits. This is required since currently valid code like this would
fail to compile otherwise:

```rust
fn foo() {
    let _ = || ();
}
```

# How We Teach This
[how-we-teach-this]: #how-we-teach-this

The term *functional* is borrowed from Java 8's similar feature. However,
"functional programming" is already a separate, well defined term, so this may
not be the best word choice.

We will need to reframe the way we talk about the `Fn*` traits and how they
interact with closures. Currently, closures are just a convenient way of
generating implementations of the `Fn*` traits, but with this change, they'll
be convenient ways of generating implementation of any functional trait.

The `Fn*` traits are still valuable and will continue to be heavily used in
canonical Rust. There are many cases where all you want is "some function with
this signature", and the `Fn*` traits are the right way to express that. In
particular, as the only variadic traits in the language, they allow us to avoid
a profusion of `Function`, `Function2`, `Function3`, etc traits, as Java is
forced to deal with. The design decision of when to use a `Fn*` trait vs a
custom trait will not change all that much from how it is now. Usage of custom
traits may become a bit more common, but, we would, for example, still use
`Fn*` traits for methods like `Iterator::position` if we were redesigning the
`Iterator` trait with this feature in mind.

The `Iterator` trait can make a good example of usage when teaching about this
feature:

```rust
fn print_items<I>(mut it: I)
where
    I: Iterator
    I::Item: Display
{
    for item in it {
        println!("{}", item);
    }
}

// an iterator with nothing in it
print_items(|| None::<i32>);

// an iterator which counts from 1 to 5
let mut n = 0;
print_items(|| {
    if n == 5 {
        None
    } else {
        n += 1;
        Some(n)
    }
});
```

# Drawbacks
[drawbacks]: #drawbacks

If closure syntax can implement any functional trait, it can become more
difficult to visually understand the types involved in a piece of code.

There is a risk that existing code fails to types infer due to this change.

# Alternatives
[alternatives]: #alternatives

Traits could have to opt-in to being functional. It's not clear what this really
buys much, however - any change that would cause a functional trait to become
non-functional would be a breaking change regardless.

[RFC 1650](https://github.com/rust-lang/rfcs/pull/1650) proposed an extension of
the closure system to allow for generic implementations of the `Fn*` traits.
This enables some of the same use cases this RFC targets. However, it would
require HRTB before functions could declare such bounds. It was postponed
pending a revamp of the trait system.

# Unresolved questions
[unresolved]: #unresolved-questions

How can we apply type hints to closures? One possibility is `impl SomeTrait`:

```rust
let thing: impl SomeTrait = |x| bar(x, 0);
```

How intelligent will the inference of the type to implement be? For example,
functions which consume iterators tend to actually take an `IntoIterator`
argument. One might expect this code to compile:

```rust
fn foo<I>(it: I)
where
    I: IntoIterator
{
    // ....
}

foo(|| None::<i32>);
```

There is a blanket implementation of `IntoIterator` for all types implementing
`Iterator`. The closure provided has a signature appropriate for
`Iterator::next` but *not* `IntoIterator::into_iter`. Would this be a type error
and have to be resolved via a type hint?

```rust
let it: impl Iterator = || None::<i32>;
foo(it);
```

Concerns have been raised about the compiler closure infrastructure's ability to
properly implement traits where the method is itself generic. In the short term,
it may be necessary to additionally require that a trait's method be non-generic
in order for it to be functional. Future advances to the compiler's
implementation will allow that restriction to be droppoed in a backwards
compatible manner.

Is the prohibition of static methods correct? There's no reason a closure
coultn't generate an implementation of e.g. `Default`. It's a bit strange for a
*closure* to be unable to *close* over any variables, but it would work. This
limitation can be relaxed in the future in a backwards compatible manner.
