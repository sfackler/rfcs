- Start Date: 2014-10-10
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

The use of traits to enable code which abstracts over implementations of some
concept is a powerful tool, but replacing inherent methods with trait
implementations introduces significant ergonomic issues, as those methods are
no longer usable unless the relevant trait is imported into scope. This RFC
proposes a system of tagging certain trait implementations as "intrinsic",
allowing their use without being forced to import those traits. This is
(modulo minor changes such as keyword reservation) implementable after 1.0 in a
backwards compatible manner.

# Motivation

Rust currently requires a trait to be imported into scope before any of the
trait's methods may be used. Newcomers to Rust are often confused by this and
ask why the language behaves this way. Couldn't the compiler figure out which
method to pick without the programming having to import the trait? While the
compiler *could* act that way, it would have some very undesirable
implications. Namely, the addition of an implementation of a trait to a crate
would break backwards compatibility!

However, this policy does add pain when working with certain types, especially
data structures. `RingBuf` is completely useless without first importing the
`Deque` trait, for example. We've been abusing the prelude to avoid this pain
with common standard library traits - imagine how annoying it would be if you
had to `use std::collection::{Collection, Mutable, Map, MutableMap};` any time
you wanted to use a `HashMap`!

Third party code can't get away with throwing things in the prelude. The best
solution currently is to implement the relevant functionality as inherent
methods on the type and then have an implementation of the trait that simply
calls through to the "real" implementations. For example, consider the
[`GenericConnection`
trait](https://github.com/sfackler/rust-postgres/blob/abd60ef1cfcd389bb6cc2cbe96a8ad99dab342b3/src/lib.rs#L1722-L1779)
in `rust-postgres`.

This solution is suboptimal as it adds a significant amount of boilerplate and
complicates the documentation.

# Detailed design

We want to avoid excess import boilerplate while avoiding the issues that come
with considering all implementations during method resolution. The key
observation here is that some trait implementations implement core
functionality of the type they're implemented for. For example, the
implementation of `Deque` for `RingBuf` would be considered a "core"
implementation - the entire point of the type is that it's a deque, and one
would never want to use `RingBuf` without the methods from that trait.

We define a new syntax for "intrinsic" trait implementations. An intrinsic
implementation of a trait can be used without explicitly importing the trait
into scope:
```rust
intrinsic impl<T> Deque<T> for RingBuf<T> {
    ...
}

mod foo {
    fn bar(d: &mut RingBuf<int>) {
        d.push_back(10); // look ma, no imports!
    }
}
```

Intrinsic implementations may only be defined in the same crate as that
implementation's type is being implemented for.

# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

What parts of the design are still TBD?
