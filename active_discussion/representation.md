# Data structure representation

## Introduction

This discussion is meant to focus on the following things:

- What guarantees does Rust make regarding the layout of data structures?
- What guarantees does Rust make regarding ABI compatibility?

NB. Oftentimes, choices of layout will only be possible if we can
guarantee various invariants -- this is particularly true when
optimizing the layout of `Option` or other enums. However, designing
those invariants is left for a future discussion -- here, we should
document/describe what we currently do and/or aim to support.

### Layout of data structures

In general, Rust makes few guarantees about the memory layout of your
structures. For example, by default, the compiler has the freedom to
rearrange the field order of your structures for more efficiency (as
of this writing, we try to minimize the overall size of your
structure, but this is the sort of detail that can easily change). For
safe code, of course, any rearrangements "just work" transparently.

If, however, you need to write unsafe code, you may wish to have a
fixed data structure layout. In that case, there are ways to specify
and control how an individual struct will be laid out -- notably with
`#[repr]` annotations. One purpose of this section, then, is to layout
what sorts of guarantees we offer when it comes to layout, and also
what effect the various `#[repr]` annotations have.

### ABI compatibilty

When one either calls a foreign function or is called by one, extra
care is needed to ensure that all the ABI details line up. ABI compatibility
is related to data structure layout but -- in some cases -- can add another
layer of complexity. For example, consider a struct with one field, like this one:

```rust
#[repr(C)]
struct Foo { field: u32 }
```

The memory layout of `Foo` is identical to a `u32`. But in many ABIs,
the struct type `Foo` is treated differently at the point of a
function call than a `u32` would be. Eliminating these gaps is the
goal of the `#[repr(transparent)]` annotation introduced in [RFC
1758]. For built-in types, such as `&T` and so forth, it is important
for us to specify how they are treated at the point of a function
call.

## Goals

- Document current behavior of compiler.
  - Indicate which behavior is "permitted" for compiler and which
    aspects are things that unsafe code can rely upon.
  - Include the effect of `#[repr]` annotations.
- Uncover the sorts of layout optimizations we may wish to do in the
  future.

## Some interesting examples and questions

- `&T` where `T: Sized`
  - This is **guaranteed** to be a non-null pointer
- `Option<&T>` where `T: Sized`
  - This is **guaranteed** to be a nullable pointer
- `Option<extern "C" fn()>`
  - Can this be assumed to be a non-null pointer?
- `usize`
  - Platform dependent size, but guaranteed to be able to store a pointer?
  - Also an array length?
- Uninitialized bits -- for which types are uninitialized bits valid?
- If you have `struct A { .. }` and `struct B { .. }` with no
  `#[repr]` annotations, and they have the same field types, can we
  say that they will have the same layout?
  - or do we have the freedom to rearrange the types of `A` but not
    `B`, e.g. based on PGO results
  - What about different instantiations of the same struct? (`Vec<A>`
    vs `Vec<B>`)
- Rust currently says that no single value may be larger than `isize` bytes
  - is this good? can it be changed? does it matter *here* anyway?

## Active threads

To start, we will create threads for each major categories of types
(with a few suggested focus points):

- Integers and floating points
    - What about signaling NaN etc? ([Seems like a
      non-issue](https://github.com/rust-lang/rust/issues/40470#issuecomment-343803381),
      but it'd be good to resummarize the details).
    - is `usize` the native size of a pointer? [the max of various other considerations](https://github.com/rust-rfcs/unsafe-code-guidelines/pull/5#discussion_r212702266)?
      what are edge cases here?
    - Rust currently states that the maximum size of any single value must fit in with `isize`
      - Can we say a bit more about why? (e.g., [ensuring that "pointer diff" is representable](https://github.com/rust-rfcs/unsafe-code-guidelines/pull/5#discussion_r212703192)
- Booleans
    - Prior discussions ([#46156][], [#46176][]) documented bool as a single
      byte that is either 0 or 1.
- Enums
    - See dedicated thread about "niches" and `Option`-style layout optimization
      below.
    - Define: C-like enum
    - Can a C-like enum ever have an invalid discriminant? (Presumably not)
    - Empty enums and the `!` type
    - [RFC 2195][] defined the layout of `#[repr(C)]` enums with payloads.
    - [RFC 2363][] offers a proposal to permit specifying discriminations.
- Structs
    - Do we ever say *anything* about how a `#[repr(rust)]` struct is laid out
      (and/or treated by the ABI)?
      - e.g., what about different structs with same definition
      - across executions of the same program?
    - For example, [rkruppe
      writes](https://github.com/rust-rfcs/unsafe-code-guidelines/pull/5#discussion_r212776247)
      that we might "want to guarantee (some subset of) newtype
      unpacking and relegate `#[repr(transparent)]` to being the way
      to guarantee to other crates that a type with private fields is
      and will remain a newtype?"
- Tuples
    - Are these effectively anonymous structs? 
- Unions
    - Can we ever say anything about the initialized contents of a union?
    - Is `#[repr(C)]` meaningful on a union?
    - When (if ever) do we guarantee that all fields have the same address?
- Fn pointers (`fn()`, `extern "C" fn()`)
    - When is transmuting from one `fn` type to another allowed?
    - Can you transmute from a `fn` to `usize` or raw pointer?
      - In theory this is platform dependent, and C certainly draws a
        distinction between `void*` and a function pointer, but are
        there any modern and/or realisic platforms where it is an
        issue?
    - Is `Option<extern "C" fn()>` guaranteed to be a pointer (possibly null)?
- References `&T` and `&mut T`
    - Out of scope: aliasing rules
    - Always aligned, non-null
    - When using the C ABI, these map to the C pointer types, presumably
- Raw pointers
    - Effectively same as integers?
    - Is `ptr::null` etc guaranteed to be equal in representation to `0_usize`?
      - C does guarantee that `0` when cast to a pointer is NULL
- Representation knobs:
    - Custom alignment ([RFC 1358])
    - Packed ([RFC 1240] talks about some safety issues)

[#46156]: https://github.com/rust-lang/rust/pull/46156
[#46176]: https://github.com/rust-lang/rust/pull/46176
[RFC 2363]: https://github.com/rust-lang/rfcs/pull/2363
[RFC 2195]: https://rust-lang.github.io/rfcs/2195-really-tagged-unions.html
[RFC 1358]: https://rust-lang.github.io/rfcs/1358-repr-align.html
[RFC 1240]: https://rust-lang.github.io/rfcs/1240-repr-packed-unsafe-ref.html
[RFC 1758]: https://rust-lang.github.io/rfcs/1758-repr-transparent.html
