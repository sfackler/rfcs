- Feature Name: ptr_freeze
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR:
- Rust Issue:

# Summary
[summary]: #summary

Uninitialized memory does not have a fixed value, and reads of uninitialized memory can very easily result in undefined
behavior. This RFC proposes APIs to allow users to *freeze* that memory, converting it into initialized memory with
an arbitrary but fixed value. It is primarily intended to be used by code dealing with implementations of the `Read`
trait.

# Motivation
[motivation]: #motivation

Consider a simple function which copies the data from a reader to a writer:

```rust
use std::io::{self, Read, Write};

pub fn copy<R, W>(from: &mut R, to: &mut W) -> io::Result<()>
where
    R: Read,
    W: Write,
{
    let mut buf: [u8; 8 * 1024] = /* ??? */;

    loop {
        let len = from.read(&mut buf)?;
        if len == 0 {
            break;
        }
        to.write_all(&buf[..len])?;
    }

    Ok(())
}
```

The reader places data into a temporary buffer, which is then passed to the writer to copy out. We do need to decide
what to initialize that buffer to, though. The simplest option is to simply zero-initialize it:

```rust
    let mut buf = [0; 8 * 1024];
```

Unfortunately, this involves quite a lot of wasted work; our reader is immediately going to overwrite all those zeroes!
It's fairly common for a reader to not contain much data, in which case the vast majority of the work done in a call
to our `copy` function would be sunk into initializing the temporary buffer. On the other hand, using a smaller buffer
will have a significant negative impact on large copies.

If we want to avoid wasting work, an alternative is to not initialize the buffer at all:

```rust
    let mut buf: [u8; 8 * 1024] = unsafe { std::mem::uninitialized() };
```

While you might think that our buffer now just contains whatever values happened to be sitting around on that region of
the stack, the `unsafe` block is a hint that it's not quite that simple. Uninitialized memory doesn't just have an
arbitary value, it has an *undefined* value. Interaction with undefined values can easily lead to undefined behavior!
Our `copy` function is safe, so we need to make sure that can't happen.

The `Read` trait is defined like this:

```rust
pub trait Read {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize>;

    // ...
}
```

The `read` method is supposed fill the provided buffer with data, and return the number of bytes written. If we want to
pass a buffer of undefined values to a reader, we need to know that it won't read any values out of `buf`, and will
correctly return the number of bytes it wrote into `buf`. For example, here are two `Read` implementations that violate
those requirements:

```rust
pub struct ExtremelyBadReader;

impl Read for ExtremelyBadReader {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        do_stuff_with_a_u8(buf[0]);
        Ok(0)
    }
}

pub struct AnotherExtremelyBadReader;

impl Read for AnotherExtremelyBadReader {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        Ok(buf.len())
    }
}
```

Both of these readers are obviously broken - using either of them is going to cause issues. However, bugginess is not
the same thing as undefined behavior! `Read` is not an unsafe trait, so we can't rely on the reader behaving properly to
keep our `copy` function from being unsound.

If only there was another way...

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC enables a sort of "middle ground" between explicitly initializing the buffer to a specific value and leaving it
with an undefined value. We can instead take a buffer of undefined values and *freeze* them into defined-but-arbitrary
values:

```rust
    unsafe {
        let mut buf: [u8; 8 * 1024] = std::mem::uninitialized();
        std::ptr::freeze(buf.as_mut_ptr(), buf.len());
    }
```

`freeze` is a directive to the compiler, telling it that it can no longer treat a region of memory as having undefined
contents. In particular, a call to `freeze` does not actually turn into any instructions in the compiled binary.

In our `copy` example, we avoid the undefined behavior concern when working with buggy readers, and avoid having to
initialize the large scratch buffer at runtime.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A new function will be added in the `core::ptr` module:

```rust
/// Freezes `count * size_of::<T>()` bytes of memory, converting uninitialized values into
/// arbitrary but fixed values.
///
/// Uninitialized memory has undefined contents, and interaction with those contents
/// can easily cause undefined behavior. Freezing the memory avoids those issues by
/// converting the memory to an initialized state without actually needing to write to
/// all of it. This function has no effect on memory which is already initialized.
///
/// While this function does not actually physically write to memory, it acts as if it
/// does. For example, calling this method on data which is concurrently accessible
/// elsewhere is undefined behavior just as it would be to use `ptr::write`.
///
/// # Warning
///
/// Take care when using this function as the uninitialized memory being frozen can
/// contain bits and pieces of previously discarded data, including sensitive
/// information such as cryptographic keys.
///
/// Frozen memory can have *any* arbitrary bit pattern, including a bit pattern that is not
/// valid for the type `T`! For example, using this function with the `bool` type is
/// unsound, since the only valid bit patterns for that type are `0b0000_0000` and
/// `0b0000_0001`.
///
/// # Safety
///
/// Behavior is undefined if any of the following conditions are violated:
///
/// * `dst` must be valid for writes.
///
/// Note that even if `T` has size `0`, the pointer must be non-NULL and properly aligned.
pub unsafe fn freeze<T>(dst: *mut T, count: usize) {
    intrinsics::freeze(dst, count);
}
```

While `freeze` is implemented as an intrinsic, it is currently equivalent to an inline assembly expression:

```rust
pub unsafe fn freeze<T>(dst: *mut T, _count: usize) {
    asm!("" :: "r"(dst) : "memory" : "volatile");
}
```

Note that this is somewhat imprecise - in particular, we have to tell LLVM that we're clobbering all accessible memory
rather than just the specified range. LLVM developers have [discussed] supporting a freeze operation directly, and we
could switch to using that in the future if/when it's implemented.

A complete implementation of the feature is available in [rust-lang/rust#58363].

[discussed]: https://lists.llvm.org/pipermail/llvm-dev/2016-October/106182.html
[rust-lang/rust#58363]: https://github.com/rust-lang/rust/pull/58363

# Drawbacks
[drawbacks]: #drawbacks

While this RFC does not add any safe APIs, it opens the door to greater exposure of not-explicitly-initialized data
to safe APIs. Frozen buffers will contain whatever data was previously in that memory location, which could potentially
include sensitive information like private keys.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Requirements:

1. Any solution must work both in static and dynamic dispatch scenarios, as `dyn Read` trait objects are very common.
2. Don't regress.
    a. Existing code interacting with the readers via the current stable API should not become significantly slower when
        the new APIs are added.
    b. Existing `Read` implementations using the current stable API should not become significantly slower when used
        with code using the new APIs.
3. `Read` implementations should not effectively require unsafe code to avoid massive performance penalties.
4. `Read` implementatiosn written before the introduction of these new APIs should not become slower after their
    introduction.

There have been previous iterations on handling the `Read` buffer initialization use case. Unlike this proposal, they
have all approached the problem by enabling `Read` implementations to unsafely opt-into working with uninitialized
buffers.

* Add a `pub unsafe trait TrustedRead: Read {}` that could be used with specialization. However, `dyn Read` trait
    objects are used quite heavily, where this approach doesn't work.
* The standard library `Read` trait currently has an [`initializer`] method, which returns an object that applies
    whatever initialization is needed for buffers passed to the reader.
* Similarly, tokio's `AsyncRead` trait has a [`prepare_uninitialized_buffer`] that directly applies any needed
    initialization.

The current approaches are slightly awkward from an API design point of view. Rust has the ability to annotate an entire
trait as being unsafe to implement, but not a single method of a trait. The `initializer` and
`prepare_uninitialized_buffer` methods are simply marked unsafe, but this isn't quite correct, as that is supposed to
indicate that the methods are unsafe to call.

This RFC improves on those approaches by moving away from `Read` implementations having to opt in to "I promise to be
careful" and towards it not mattering if the implementations are careful or not.

[RFC 837]: https://github.com/rust-lang/rfcs/pull/837
[uninitialized memory]: https://internals.rust-lang.org/t/uninitialized-memory/1652
[zero memory]: https://github.com/rust-lang/rust/pull/23668
[initializer]: https://github.com/rust-lang/rust/pull/42002

[`initializer`]: https://doc.rust-lang.org/std/io/trait.Read.html#method.initializer
[`prepare_uninitialized_buffer`]: https://docs.rs/tokio-io/0.1.11/tokio_io/trait.AsyncRead.html#method.prepare_uninitialized_buffer

# Prior art
[prior-art]: #prior-art

Memory unsafe languages like C and C++ simply use uninitialized memory directly in these contexts and rely on the APIs
to behave properly to avoid undefined behavior.

Memory safe languages like Java require buffers to be fully initialized before use.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

The freeze intrinsic is the minimal API required to enable the use cases discussed above, but it is not the most
convenient construct. As we gain more experience, we could add methods to handle the common use cases more easily:

```rust
impl Vec<u8> {
    /// Like `resize`, except that it freezes new capacity rather than initializing it to a specific value.
    pub fn resize_frozen(&mut self, new_len: usize) { ... }
}

impl<T> MaybeUninit<T> {
    /// Create a new `MaybeUninit` in a frozen state.
    pub const fn frozen() -> MaybeUninit<T> { ... }
}
```

While the use cases described above revolve entirely around frozen `u8` values, the same concept can apply to a broader
class of values. We could add an auto trait to represent this concept so that methods like `Vec::resize_frozen` can be
safe and support more than one type:

```rust
// NB: this won't work exactly as written since uninhabited enums can't be Pod
pub auto trait Pod {}

unsafe impl<T> Pod for *mut T {}
unsafe impl<T> Pod for *const T {}
unsafe impl Pod for u8 {}
unsafe impl Pod for u16 {}
// ...

impl<T> Vec<T>
where
    T: Pod,
{
    pub fn resize_frozen(&mut self, new_len: usize) { ... }
}
```
