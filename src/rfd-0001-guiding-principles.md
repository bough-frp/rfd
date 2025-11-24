# Guiding Principles for Bough Design

Bough aims to be the go-to Functional Reactive Programming library in
Rust, and there are a number of ideals we want to strive for:

## Strong Adherence to the Sodium FRP Denotational Semantics

Software needs to be more composable. Instead of using LLMs to
generate reams of new code in existing languages lets build better
tools and languages that are expressive enough that we don't need to
use LLMs.

Having a library that lets you leverage compositionality and a direct
path to creating a functional-core imperative shell architecture for
complex event driven programs in Rust would be a strong step in this
direction.

This means that first and foremost Bough needs to correctly implement
the Sodium FRP denotational semantics (modulo API names).

## Safe and Idiomatic Rust API

Rust is a great foundation to build systems software in, but sometimes
it's not the best tool for every job. Building a higher level
abstraction like a library for Functional Reactive Programming that is
deeply integrated with Rust seems like it could give us the best of
both worlds.

Similarly, Sodium is a well-design minimal FRP library but the Rust
port is clearly designed to conform to the standard Sodium API and
doesn't do much to take advantage of Rust's type system to provide a
safe and ergonomic API.

Bough's public API surface should be entirely safe Rust and take
advantage of Rust's type system to the fullest to provide an ergonomic
and idiomatic experience writing FRP in Rust. Writing correct FRP code
in Bough should be easy and obvious, violating the expectations of the
Bough FRP engine should as be difficult and awkward as possible.

## As Performant as Possible

FRP is a higher-level abstraction and as such we expect to sacrifice
some performance for the sake of safety and ergonomics. But Bough is
still a Rust library so let's not leave any obvious performance wins
on the table.

## Usable as a Library

Bough is first and foremost meant to be used as a library from within
Rust.

## Easy to Build More Tooling On

Once we have a solid base to work from building a more full-featured
FRP based app framework on top of Bough is an explicit goal.
