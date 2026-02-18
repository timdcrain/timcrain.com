+++
title = "impl Trait"
date = 2026-02-17T22:32-08:00

[taxonomies]
tags = ["rust"]

[extra]
tagline = "Existential types in Rust"
+++

I'm a little embarrassed that I've been writing Rust for the last four years and
just now realized that `fn foo(arg: impl Trait)` is _not_ a function accepting a
trait object but is in fact exactly the same as `fn foo<T: Trait>(arg: T)`.
[^dyn-trait] It must have been a recent change, right? Oh,
[2018](https://blog.rust-lang.org/2018/05/10/Rust-1.26/#impl-trait). Anyways,
why was this syntax added? I don't think that this syntax is enough of an
improvement [^syntax] to add the complexity of a second way of doing things.

It turns out that there were a lot of reasons, [^rfc] but one of the main
reasons is symmetry with `impl Trait` in the return position, i.e. `fn foo() ->
impl Trait`. This is more interesting because it is not equivalent to `fn foo<T:
Trait>() -> T`. In fact, it is the complete opposite: a function returning an
`impl Trait` can only return a single concrete type.

In type theory this is an **existential** type, where the caller only knows that
a type _exists_ that satisfies the given abstract type. [^existential] In Rust,
this allows the compiler to use the returned type as if it was a concrete type
(e.g. static dispatch, no boxing) while hiding that from the caller. The `impl
Trait` in the argument position is a **universal** type, where the callee must
accept _all_ types that satisfy the abstract type.

Are there any real-world uses for `impl Trait` as a return type? There is one
very prominent (but not obvious) example---async functions actually return an
`impl Future<Output=T>`. Because futures have unique, un-writable types, they
can only be returned either as `impl Future<Output=T>` or as `Box<dyn
Future<Output=T>>`, and so the `impl Trait` feature allows Rust to have
stack-allocated futures.

Anyways, I doubt I will be writing functions that return an `impl Trait` very
often, but it's a really neat example of how Rust enables zero-cost
abstractions.

[^dyn-trait]: I blame the similarity to `dyn Trait`. `fn foo<T: Trait>(arg: T)`
    is so obviously generic while `fn foo(arg: impl Trait)` (generic) looks very
    similar to `fn foo(arg: Box<dyn Trait>)` (not generic). I think I just saw
    `Trait` in the argument position and assumed it was the latter.

[^syntax]: Actually, I much prefer the more verbose syntax. It feels much more
    intentional.

[^rfc]: See [RFC 1951](https://rust-lang.github.io/rfcs/1951-expand-impl-trait.html)
    for a more in-depth discussion.

[^existential]: In Rust, a function that returns an existential type can only
    return a single concrete type, but that is not a general fact about
    existential types.
