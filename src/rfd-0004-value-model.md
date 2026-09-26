# Value Model: Linear Streams, Explicit Sharing, and Where Clone Happens

**State:** discussion · **Authors:** Zefira Shannon, Claude

Sodium hands values around by reference in garbage-collected
languages, and `sodium-rust` requires `Clone` on everything and clones
at every fan-out. Bough puts `Clone` only where a value is genuinely
duplicated, lets non-`Clone` values flow through streams, and makes
the user confront the point where duplication happens. This RFD is
also where the distinction between shared nodes and intermediate
combinators that the `bough` repository's README promises becomes
concrete.

## By Value Through Streams, By Reference From Cells

Stream functions take their event by value and return owned
values: `map` takes `A` and returns `B`, `snapshot` takes `A` and
`&B`, `merge` takes `A, A`, and `filter` takes `&A` since it does not
consume. The identity function is `|a| a`, not `|a| a.clone()`. Cell
functions take references: `map_cell` and `lift` take `&A`,
`accumulate` takes `A, &S` and returns `S`, and `scan` takes `A, &S`
and returns `(B, S)`. Stream listeners take `A`; cell listeners take
`&A`.

A reference to a cell's value lives for exactly one call. A stored
closure is `'static`, and the reference it receives has an anonymous
lifetime bounded by the call, so keeping it is a compile error rather
than a rule. Keeping a cell value means cloning it, and the clone is a
snapshot as of that instant. The one way to see a later change through
a kept value is interior mutability inside the value, and that breaks
every FRP system identically, because a listener that mutates through
an `Rc<RefCell<_>>` rewrites every past snapshot. That is a user
violation, not something this model introduces.

## Streams Are Linear; Fan-Out Is Explicit

A `Stream<A>` is move-only and has exactly one consumer. Every
constructor consumes it. Using it twice is a compile error, and the
fix is to say so: `share(b)` returns a `Shared<A>`, which is `Copy`,
requires `A: Clone`, and clones on every read. Both implement
`Source`, a trait with an associated `Event` type the way `Iterator`
has `Item`, so every constructor accepts either through one trait, and
the dependency is compiled into the node at construction: a linear
dependency takes the event out of the slot, a shared one clones it.
`listen` is the exception: it accepts the two node types only, through
`Node`, which adapter types do not implement, so no chain can be
handed to I/O code ([RFD 2](./rfd-0002-strong-io-separation.md)).

Linearity is what lets `hold` and `merge` drop their `Clone` bounds.
The hold is the sole consumer and moves the event into its
committed value; merge moves whichever input fired through its
function. The `Clone` points that remain are the operations that
genuinely duplicate a value:

- `share`;
- `map_to`, which emits one value repeatedly;
- `steps` and `steps_with_current`, which emit a value the cell also
  keeps;
- a caller of `sample` who clones; `sample` itself returns a reference
  and has no bound.

The listeners `listen_steps` and `listen_cell`, the I/O forms of the
two stream views ([RFD 2](./rfd-0002-strong-io-separation.md)), read
the committed value after commit by reference and need no `Clone`.

We considered a single `Copy` stream token with a build-time panic on
a second consumer. It finds the bug on first run; the move-only token
finds it at compile time, which is the point of the model. The cost is
that a linear stream inside a value is unreachable for derivation,
because cell values are read by reference and a function on
`Cell<Item>` cannot move `item.clicks` out. Any stream that travels
inside a value must be shared, which is the confrontation we want. I/O
code receives a linear stream only as a by-value event, and since
listeners have no graph access, it attaches a listener only after
`send` returns: dynamic wiring from I/O is receive, then wire.

A cell can hold linear tokens directly, `hold(b, init)` over a stream
of streams, and `switch_stream` may switch over such a cell, exactly
one switch per such cell, checked when a second is constructed. This
is the `Clone`-free path for the main dynamic pattern: `construct`
builds a screen, a hold keeps the current one, one switch reads its
events. Requiring `Shared` for every switched stream would put a
`Clone` bound on every dynamically constructed event type, which undoes
half the benefit.

## Chains Fuse, Iterator Style

Nothing can observe the values between the adapters of a linear stream,
so the adapters fuse the way iterator adapters do. `map`, `filter`,
`filter_map`, `map_to`, `snapshot`, `gate` and `once` are adapters: each
returns its own type, such as `Map<S, F>`, all of them `Source`, and
none takes a build context. A chain is a linear sequence of adapters
with no materializer. `Source` carries its event as an associated
type, `Event`, rather than a type parameter: an adapter such as
`Map<S, F>` cannot implement a generic `Source<A>`, because `A` would
appear only in its bounds (E0207), which a stub of the API surface
confirmed. A chain becomes one node with one
monomorphized closure when something materializes it: `hold`,
`accumulate`, `accumulate_mut`, `scan`, `share`, `node`, `merge`,
`or_else`, `split`, `defer`, `construct`, `switch_stream`, and on
cells `map_cell`, `lift` and `switch_cell`. Those take `b`.

```rust
input.map(f).filter(p).snapshot(c, g).hold(b, 0)
```

A chain cannot be stored in a value or returned from build until it is
materialized, like an iterator before `collect`; `node(b)` materializes
a chain as a linear stream with an identity of its own. The marking
walk and the dependents lists see one node per chain, and the
per-adapter cost is a direct call, so a chain costs what the imperative
baseline costs. The alternatives were one node per adapter, a virtual
call and a slot per adapter, and boxed adapters appended to a chain
node, which removes the bookkeeping but keeps a virtual call per
adapter. Fusion is the iterator-like API the `bough` README promises,
and it is the cheap version of a compiled graph for the one case that
dominates real graphs; the node graph itself stays an interpreter
([RFD 5](./rfd-0005-transaction-protocol.md)).

`once` carries its state inside the fused closure and sets it during
evaluation. A chain evaluates at most once per transaction, so
this is indistinguishable from updating at commit.

## Cells

A hold is the stateful cell: it moves its event into its committed
value at commit, and an accumulator is stateful the same way.
`map_cell`, `lift` and `switch_cell` are read-through: they compute
from their inputs' current values when read and memoize the result, so
the function runs zero times if the cell is never read and at most
once per step per reader. A cell read once per frame while its input
steps a thousand times per frame costs one call.

A read-through cell is a node with an identity, and its inputs are its
dependencies: it steps whenever an input steps. The marking walk
reaches it through the dependents lists like any stream node, but
marking it costs one flag and evaluates nothing. At commit the memo of
every marked read-through cell is cleared; after commit its listeners
run, and the first read computes the new value. Nothing less would
make `listen_cell` on a read-through cell fire at all, since a listener
needs a step to fire on. The first draft kept read-through cells
outside the dependents graph and compared version counters on read,
and under it the flagship example in
[RFD 2](./rfd-0002-strong-io-separation.md), a listener on a
`map_cell`, would have fired once at registration and never again.

The memo is a `std::cell::OnceCell<A>`. It hands out `&A` from a
shared borrow, which is what lets two samples compose in one
expression, and it can only be cleared through `&mut`, which commit
has and no reader does: holding a sampled reference across a `send` is
a borrow error, not a rule. A stub confirmed both. `OnceCell` is `Send`
when `A` is, which is all `Graph<Threaded>` needs, since every entry
that drives the graph takes `&mut self` and contention cannot occur;
the first draft named a `Cell`-style slot and a mutex, and neither can
return a reference, because `Cell` has no `borrow` and a mutex guard
dies at the end of `sample`.

Functions must be pure. The engine calls a read-through function at
most once per step per reader and not at all if the cell is never
read; a `steps` view of the same cell computes its own value during
evaluation, so a function may run twice for one step. We considered
eager evaluation at commit, which matches Sodium's call pattern and
makes sample a single load. It is never better than read-through by
more than a flag check, and it is unboundedly worse for the high-rate
shape read by a slow observer, which is one of the three workloads.

`lift` takes a tuple of cells, arities two to six as Sodium ships them,
and one function over references to all of them:
`(price, quantity).lift(b, |p, q| p * q)`. There is no binary method to
chain, because `a.lift(b, x, f).lift(b, y, g)` composes two functions
rather than lifting three cells, and expressing a three-argument
function through it needs an intermediate cell that clones two inputs
on every read. Sodium's `apply`, a cell of functions applied to a
cell, is `(cf, ca).lift(b, |f, a| f(a))` with the cell holding
`Fn(&A) -> B`, since cell values are read by reference. Its
simultaneity rule, the semantics' `knit`, is what marking gives: two
inputs stepping in one instant mark the lifted cell once.

`switch_cell` is read-through on read, `at (SwitchC c) t` is
`at (at c t) t`, two pointer chases and no memo. It has state all the
same: the inner it currently depends on. The semantics'
`steps (SwitchC c t0)` splices in every step of the selected inner, so
the node is a dependent of that inner, relinked at commit whenever the
outer steps, and it is marked at creation and at every switch instant
even when the new inner is quiet
([RFD 5](./rfd-0005-transaction-protocol.md)).

The stream views of a cell are `steps` and `steps_with_current`,
Sodium's `updates` and `value`, materializers on `Cell` that carry the
book's warning ([RFD 2](./rfd-0002-strong-io-separation.md)). Both
require `A: Clone`, since a hold keeps its value and the stream needs
one of its own, and both are stream nodes evaluated in order like any
other: a step in the cell's inputs is one event carrying the
post-instant value, computed during evaluation from the inputs'
post-instant values. `steps_with_current` also fires at its creation
instant with the post-instant value, which is the semantics'
`coalesce (flip const) ((t0, a) : sts)`: a creation and a step in one
instant are one event carrying the new value. The listeners
`listen_steps` and `listen_cell` are the I/O side of the same two
views.

## Accumulation

`accumulate(b, init, f)` with `f: Fn(A, &S) -> S` is the semantics'
knot: `let c = hold(init, snapshot(f, s, c))`, a hold whose input
snapshots the hold itself. The cell being snapshotted is the result,
which is what makes an accumulator a legal way to close a loop: it is
already a loop through a hold. Written as a straight line over some
other cell it would be a different operator, one that never reads its
own state, and it would typecheck. On collections it is quadratic:
every event clones the collection to push one element.

`accumulate_mut(b, init, f)` takes `f: FnMut(A, &mut S)`. The `&mut S`
is the asymptotic fix: a `Vec` accumulator is a push. Running `f` at
commit, after every reader in the transaction has seen the
pre-transaction state, is what makes the mutation unobservable: every
reference to the state is scoped to one call, and the mutation happens
when none exists, so observationally it is Sodium's `accum`.

Because the new state does not exist until commit, an in-place
accumulator has no stream view: `steps` and `steps_with_current` on its
cell are a build-time panic, since the event they would carry has no
value during evaluation. `listen_steps` and `listen_cell` read the
committed state after commit and work as usual. This is the one
restriction the in-place form carries, checked at build time where a
missing value cannot hide; a type-level split, a distinct `State<S>`
token that every cell-reading operation accepts through a trait, stays
available if the panic bites in practice. Dropping in-place
accumulation would leave an asymptotic cliff against the performance
bar for any workload that accumulates, and persistent collections cost
roughly ten times a `Vec` push. Dropping `accumulate`, the semantics'
form, would lose a legal and common stream view.

`scan(b, init, f)` with `f: Fn(A, &S) -> (B, S)` is Sodium's `collect`:
at each event it emits `B` and holds `S`. It stays derived, since its
output is needed during evaluation while its state change waits for
commit. `Iterator::scan` is the nearest Rust name and no more than
that: it takes `(&mut S, A)` and ends the iteration on `None`, while a
stream has no end and every event yields one output.
