# Guiding Principles for Bough Design

**State:** discussion · **Authors:** Zefira Shannon, Claude

Bough aims to be the go-to Functional Reactive Programming library in
Rust, and there are a number of ideals we want to strive for:

## Strong Adherence to the Sodium FRP Denotational Semantics

Software needs to be more composable. Instead of using LLMs to
generate reams of new code in existing languages we should build better
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
Bough FRP engine should be as difficult and awkward as possible.

## As Performant as Possible

FRP is a higher-level abstraction and as such we expect to sacrifice
some performance for the sake of safety and ergonomics. But Bough is
still a Rust library so we should not leave any obvious performance
wins on the table.

## Usable as a Library

Bough is first and foremost meant to be used as a library from within
Rust.

## Easy to Build More Tooling On

Once we have a solid base to work from building a more full-featured
FRP based app framework on top of Bough is an explicit goal.

## Policies

These are standing rules rather than decisions. They apply to every RFD
and every increment, and changing one is a change to this document.

### The Semantics Are the Test

The executable denotational semantics in the Sodium repository
(`denotational/Reactive/Sodium/Denotational.hs`, version 1.1) are
ported to Rust as an oracle over lists of time-stamped values, the
`bough-oracle` crate. Every primitive is property-tested against the
oracle with random programs and random input events, comparing
events and steps at every instant. The port is cross-checked once
against the fixed vectors in `denotational/sodium.hs` and
`common-tests/SemanticTests.hs`; GHC does not run in CI. Behaviour the
semantics cannot express (listeners, roots, collection, runtime inputs)
gets its own property tests with random observation patterns, and the
public live-node count is the leak assertion.

The semantics are vendored into `bough-oracle`: the Haskell, and the
markdown rendering of the accompanying document, under their BSD
licence, so a test cites the section it holds the engine to and stays
correct after the internals it was written against are replaced. The
random-program generator produces loops and diamonds through loops from
its first version, because every correctness problem the Bevy port's
research met involved a loop, and the `lift2` shape it filed as
`sodium-rust#52`, a cell held through a loop lifted together with
something upstream of itself, is a fixed test written against the
specification. The engine and the oracle also run on `wasm32-wasip1`
under wasmtime once the engine exists ([RFD 7](./rfd-0007-targets.md)).

Exact fidelity means we inherit the corners. `listen_steps` fires on a
step to an equal value. `switch_cell` emits a step at creation and at
every switch, even when the new inner is quiet. `switch_stream` uses
the old stream at the switch instant while `switch_cell` uses the new
cell's
post-instant value. `merge` takes a combining function, called as
`f(left, right)` when both streams fire in one instant; `or_else` is
the left-biased one. Changing any of these is a semantics change and
needs its own RFD, not an implementation choice.

### Test Affordances

Five, and nothing beyond them, because test-only introspection is how
it leaks into the engine: a runtime setting that collects after every
transaction; a debug dump of the graph behind a cargo feature; node ids
allocated deterministically in creation order, so a failure reproduces;
the public live-node count; and a seeded setting that shuffles listener
dispatch order and the evaluation order of independent nodes, so the
property tests run under random orders and a user's tests surface
order-dependent listeners. The fifth is what makes the claim that order
cannot matter a test that can fail rather than a sentence.

### What Fast Means

Three benchmark shapes, each with a hand-written imperative baseline,
timed by criterion by hand:

- UI: ten thousand nodes, ten switches deep, one input per
  transaction, ten listeners touched. The baseline is an observer
  registry with boxed callbacks, because dynamic wiring is intrinsic
  to the shape.
- Frame: a thousand inputs per transaction into ten thousand nodes
  with most of them affected. The baseline is a loop over the values.
- Shallow: one input through three adapters, plus a variant with a
  `share` in the middle so one real node hop is measured. The baseline
  is plain function calls.

The bar is a factor of three against the baseline on realistic per-node
payloads of 50 to 100 nanoseconds of the user's own work. Overhead on
trivial payloads is reported alongside as information, not as a bar; no
design with transactions, generation checks and a dirty walk gets under
roughly ten times a tight loop on trivial payloads, and the answer to
that is the usual FRP advice, cells of collections rather than
collections of cells. `iai-callgrind` runs the same shapes in CI as a
regression gate on instruction counts. It does not measure the bar,
because instruction counts do not track wall-clock across cache
effects. Code size on wasm32 is reported as information in the same
way, and has no gate ([RFD 7](./rfd-0007-targets.md)).

### Targets

Native with `std`, bare metal on Cortex-M without it, the web through
`wasm32-unknown-unknown` and wasm-bindgen, and Bevy as a host are
first-class targets. A decision that breaks one is a decision to
revisit, and continuous integration checks the core for each of them on
every push ([RFD 7](./rfd-0007-targets.md)).

### Dependencies

The engine crate has zero runtime dependencies in its default feature
set. A feature may add one when it is the seam an ecosystem already
implements, as `critical-section` is for bare metal. `proptest`,
`criterion` and `iai-callgrind` are development dependencies.

### Naming

No single-letter suffixes and no abbreviations, except where Rust
precedent exists (`*_mut`, `try_*`, `filter_map`). We can afford to
spell words out. Sodium's names are kept where they are good and
replaced where they are poor; the translation table lives in the
crate-level documentation and in the explanation book.

### Documentation

[Diátaxis](https://diataxis.fr/). Reference is rustdoc. Tutorials,
how-to guides and explanation live in an mdBook at `book/` in the
`bough` repository, published to GitHub Pages on every `v*` tag.

### Records

RFDs carry a `State` line using Oxide's states: `prediscussion`,
`ideation`, `discussion`, `published`, `committed`, `abandoned`. A
decision is recorded when something costly to undo is about to depend
on it. Trade-offs and rejected alternatives go into the section they
belong to, not into a change history.
