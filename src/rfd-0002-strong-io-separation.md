# Strong Separation of Constructing FRP and I/O

**State:** discussion · **Authors:** Zefira Shannon, Claude

One flaw in the Sodium API is that it is possible to use I/O bridge
APIs inside of FRP logic and doing this seriously compromises the
correctness of the FRP engine.

We can do better than this in Rust.

## Flaws in Sodium's API Design

The first problem with the Sodium API is that it is so minimal that
some methods that should only be used for interfacing with the I/O
edge ("World of I/O" in Sodium parlance) are directly on the `Stream`
and `Cell` structs and so they show up in the documentation right next
to all the core FRP primitives. Creating an input FRP
struct requires first creating the input edge struct (an I/O API!), so
from the very jump the Sodium API is _requiring_ the user to mix the
usage of I/O and FRP code.

The second major problem is that the `Transaction` concept is used for
both constructing FRP and in the I/O edge. From an implementation
perspective this makes sense because a transaction is needed. From an
API perspective it again muddles the distinction between the FRP and
I/O. This dual-usage of `Transaction` also doesn't help guide the user
towards the standard pattern of building all FRP up-front and then
"turning the crank" and just sending events to the constructed graph.

Because of these flaws and because Sodium in Rust is a port of the C++
implementation, there is no attempt to leverage unique Rust features
like lifetimes to prevent the use of I/O edge APIs inside FRP because
the design is fundamentally unsuited to doing so.

## Two Types: Build and Graph

The key insight is that we can (and should!) actually represent the
difference between "FRP build time" and "I/O event sending time" in
the API. The denotational semantics already draw this line: `hold`,
`sample`, `value` and `switchC` live in the `Reactive` monad, the pure
operations do not, and `Execute` is the only way to run construction
at event time. We map `Reactive` onto a `Build` context that
every node-creating operation requires and that also carries `sample`,
and we put sending, listening, sampling from outside, and garbage
collection on a `Graph`, which only exists once the build closure has
returned.

The invariant this buys is best stated as one sentence: **the graph is
entered from exactly one place at a time.** Graph code, meaning node
functions and `construct` closures, never holds a `Graph`. During a
transaction the engine holds the only `&mut Graph`, and `Build` has no
`send` and no `listen`. Every public
entry point on `Graph` also checks a flag saying whether a transaction
is in progress, so even a `Graph` smuggled into graph code inside an
`Rc<RefCell<_>>` fails deterministically at the entry, not somewhere
downstream.

What this does not, and cannot, prevent is asynchronous effects. A
node function may capture a channel sender, or a `Remote`
([RFD 6](./rfd-0006-io-edge.md)), and hand a value to another thread
that later sends it into the graph. That send lands in a later
transaction, which makes it I/O by definition rather than a hole in
the wall; the direct case, a `Remote` used from graph code on the
driver thread, is an error from the start of a transaction until its
listeners run, and [RFD 6](./rfd-0006-io-edge.md) gives the guard.
The synchronous guarantee is the one that matters for correctness,
because it is the one that keeps a transaction a pure function of its
inputs.

```rust
use std::{cell::RefCell, rc::Rc};

struct Click;
#[derive(Trace)]
struct Edge { clicks_in: Input<Click>, label: Cell<String> }

let ui = Rc::new(RefCell::new(Ui::new()));

let (mut graph, edge) = Graph::build(|b| {
    let (clicks, clicks_in) = b.input::<Click>();
    let count = clicks.accumulate(b, 0u32, |_, n| n + 1);
    let label = count.map_cell(b, |n| n.to_string());
    Edge { clicks_in, label }         // whatever build returns is the edge, and the root set
});

let _l = graph.listen_cell(edge.label, {
    let ui = ui.clone();
    move |s| ui.borrow_mut().set_text(s) // fires now, then on every step
});
graph.send(edge.clicks_in, Click);                                   // one transaction
graph.transaction(|tx| { tx.send(edge.clicks_in, Click); });         // several sends, one instant
```

The example compiles against a stub of this API. `graph` is `mut`
because the operations that drive the graph take `&mut self`; `sample`
and `try_sample` take `&self`, so two samples compose in one
expression (see Reading a Cell). The listener owns a clone of the
shared UI state because a listener is `'static`.

### Tokens

Streams and cells are inert tokens: an index, a generation, and a
graph id. They have no method that creates a node without a `Build`
context, so I/O code can hold and pass them around but cannot
build with them. `Cell<A>`, `Input<A>` and `Shared<A>` are `Copy` for
every `A`, through hand-written impls rather than derives, so
`Cell<String>` is `Copy`. `Stream<A>` is move-only, for the reasons
[RFD 4](./rfd-0004-value-model.md) gives. `Input<A>` is `Copy` because
several I/O sources may legitimately drive one input, and because
inputs travel as data (see Inputs below). The graph id is checked
wherever a token meets a graph, which is at materialization for graph
code and at every I/O entry point; a token from one graph used in
another is a build-time panic in graph code and an error from the I/O
API. Dropping the graph id would save four bytes per token and buy
undefined behaviour, in the logical sense, across graphs. Four bytes
and one compare is nothing.

### Chains and Nodes

Constructors are methods on the tokens, and they come in two kinds.

Adapters transform events and take no context: `map`, `filter`,
`filter_map`, `map_to`, `snapshot`, `gate` and `once`. They live on
the `Source` trait, and each returns its own type, `Map<S, F>`,
`Filter<S, P>` and so on, which is itself a `Source`. "Adapter" names
both the operation and the type, as it does for `Iterator`. Nothing is
allocated and no node exists yet. `Source` has an associated `Event`
type rather than a type parameter, the way `Iterator` has `Item`, so
`Source<Event = Click>` reads as a source of click events; a generic
`Source<A>` cannot be implemented by an adapter type at all, because
`A` would appear only in the bounds (E0207). A chain is a linear
sequence of adapters with no materializer, and it is itself linear:
using it twice is a compile error.

Materializers create exactly one node from a chain and take the build
context: `hold`, `accumulate`, `accumulate_mut`, `scan`, `share`,
`node`, `merge`, `or_else`, `split`, `defer`, `construct`,
`switch_stream`, and on cells `map_cell`, `switch_cell`, `steps` and
`steps_with_current`, plus `lift` on a tuple of cells. The adapters
between two nodes fuse into that node's closure;
[RFD 4](./rfd-0004-value-model.md) records the fusion. `Build` itself
has methods only for the things that start from nothing: `input` and
its variants, `constant`, `never`, `cell_loop` and `stream_loop`, plus
`depends` ([RFD 3](./rfd-0003-memory-model.md)).

```rust
input.map(f).filter(p).snapshot(c, g).hold(b, 0);   // three adapters, one node
(count, other).lift(b, |n, m| n + m);
```

We considered putting every constructor on `Build`,
`b.hold(0, b.filter(b.map(s, f), p))`, which nests inside out, and a
chained builder type borrowing `b` for one expression, which is a
second surface to keep in sync with the first. Methods on the tokens
read like Sodium, the chain runs left to right, and the move of `self`
is the linearity check made visible.

### Inputs

`b.input::<A>()` returns a stream and the `Input<A>` token that drives
it. `b.input_coalescing(f)` is for inputs that may be sent more than
once in a transaction, with `f: Fn(A, A) -> A` taking both values by
value, first send on the left. Its event type comes from the first use
or from an annotated closure; a bare `|x, y| x + y` fixes nothing, and
the turbofish takes two parameters, `::<u32, _>`, since the closure
type is the second. `b.input_cell(init)` and
`b.input_cell_coalescing(init, f)` return a cell and its token, a hold
over an input. The cell forms are one line over the stream forms and
exist because an input that is state is common enough to deserve a
name.

Two sends to a non-coalescing input in one transaction are an error in
both build modes. This is a rule about one instant, not about who
holds the token, so no token discipline could make it a compile error:
one holder sending twice violates it just as two holders do. Silent
last-wins would make production behave differently from every test
that ever ran.

Inputs may be created anywhere `Build` is available, including inside
`construct`, and the token flows to I/O code as data, through a
listener on the value that carries it. The alternative, declaring
every input as a parameter of the build closure, cannot express an
input created at runtime for a dynamically constructed component, and
multiplexing every dynamic input through one keyed input such as
`Input<(ItemId, Edit)>` pushes routing logic into every component.

An input may also be fed by an input slot, a `static` the writer owns,
connected with `b.connect(input, &SLOT)` and drained by `pump`; that is
the path from an interrupt handler ([RFD 7](./rfd-0007-targets.md)).

### Listeners and Handles

Listeners are `FnMut(A)` for streams and `FnMut(&A)` for cells and
nothing else: no sample, no send, no listen from inside a listener.
State a listener needs is snapshotted into the graph, where the
dependency is visible and glitch-free. Listeners run after commit,
from the finished slots, so a panicking listener leaves a consistent
graph, and no listener is ever registered mid-transaction. Sodium lets
listeners sample and delivers to them during the transaction; removing
both removes the suppress-earlier-firings flag from the engine. The
order of listeners within a transaction is the evaluation order of
their streams, ties by registration order, and is documented as not
something to rely on.

`listen`, `listen_cell`, `listen_steps` and `anchor` return a
`Listener` or an `Anchor`. A handle borrows nothing from the graph: it
shares a flag with its node, and dropping it flips the flag, so the
graph can be driven while handles are held and a handle can be dropped
inside a listener. The handles carry the graph's mode,
`Listener<M = Local>` and `Anchor<M = Local>`, because that flag is a
counted cell in `Local` and an atomic in `Threaded`
([RFD 6](./rfd-0006-io-edge.md)). `unlisten()` exists for symmetry
with Sodium, and `keep()` turns a handle into an app-lifetime root
without a struct to hold it. [RFD 3](./rfd-0003-memory-model.md) has
the rest.

`listen` accepts materialized nodes only, `Stream<A>` or `Shared<A>`,
never a chain, through a `Node` bound that adapter types do not
implement. A listener with a pre-filter would be FRP logic constructed
after build, and `snapshot` and `once` in I/O code would be the first
crack in the wall. A linear stream is moved into `listen`, so it can
be listened to once; a shared stream any number of times; a cell any
number of times through `listen_cell` and `listen_steps`.

Sodium's `updates` and `value` are its operational primitives, filed
under `Operational` because they expose a cell's steps. The book puts
the warning plainly in section 8.4: "To protect the idea of a
continuously varying cell, a true FRP system must ensure that changes
in a cell's value aren't observable." Both exist here, in graph code
and in I/O code. In graph code, `c.steps(b)` is `updates` and
`c.steps_with_current(b)` is `value`, materializers on `Cell` that
return a stream, and their documentation carries that warning: a
stream of a cell's steps observes how the cell was built, not only
what it holds, and belongs in operational code such as sending a cell
over a wire. From I/O code, `listen_steps` is `updates` and
`listen_cell` is `value`, listeners that deliver the value by
reference. We first moved the stream views to `Graph` alone, so that
no stream view of a cell existed in graph code. That made graph code a
strict subset of the semantics: a read-through cell has no feeding
stream to keep, and the nearest substitute, a snapshot on its inputs'
streams, is one instant stale, because `snapshot` reads a cell as it
was before the instant. Restoring the stream views keeps the semantics
whole and puts the warning where Sodium put it, on the primitives
themselves; what they cost is recorded in
[RFD 4](./rfd-0004-value-model.md).

### Outputs

Any stream or cell token can be listened to from I/O, with the
once-versus-many rule above, and whatever the build closure returns is
the permanent root set ([RFD 3](./rfd-0003-memory-model.md)). We
considered a distinct `Output<A>` type that build must produce
explicitly. Dynamic outputs travel inside values, and an `Output`
wrapper would have to be applied inside every `construct` closure; the
build return value already says "these are the edges".

There is no queue type in the core. A loop that wants to pull
events instead of reacting to them writes a listener that pushes
into its own collection, and the adapter crates for specific runtimes
provide that over their own channel types
([RFD 6](./rfd-0006-io-edge.md)). We drafted a core `Mailbox<A>` and
dropped it: it needed a capacity policy, a drain contract and drop
semantics of its own, for something every runtime already has.

## Runtime Construction

Nothing can add logic after build. Runtime construction goes through
`construct`, which is the semantics' `Execute`:
`s.construct(b, |b, a| ...)` runs the closure at each event with
a fresh `&mut Build`, and its results are ordinary events. A
constructed screen goes into a hold that a `switch_stream` or
`switch_cell` reads, and a token created inside the closure flows out
to I/O code as data, as Inputs describes. Plugin-style late logic is a
cell of plugins and a switch. Tests build one graph each.

Inside `construct`, `sample` returns the value the cell had at the
start of the transaction, the semantics' `at c t`. That value is fixed
for the whole transaction, so the read imposes no ordering and cannot
glitch, which is also why cell reads are never dependencies in the
evaluation order ([RFD 5](./rfd-0005-transaction-protocol.md)). A
hold created inside `construct` starts at its initial value and picks
up an event in the same transaction if its input has one, as the
semantics' `Hold a s t0` with `t >= t0` requires.

## Loops

FRP allows a limited form of cycles in the constructed graph. Declare
and close are separate, and flat:

```rust
let (block_number, block_number_loop) = b.cell_loop::<BlockNumber>();
let (retry_count, retry_count_loop) = b.cell_loop::<RetryCount>();
let parts = build_transfer(b, block_number, retry_count, ...);   // forward tokens travel anywhere
block_number_loop.close(b, parts.block_number);                  // any cell, subject to the rule below
retry_count_loop.close(b, parts.retry_count);
```

A cell loop closes with any cell and the forward token becomes that
cell. A stream loop returns a linear forward stream and closes with
any chain. The cycle rule is about paths, not about the closing
expression: every path from the definition back to the forward token
must pass through a `hold`, an accumulator, a `split` or a `defer`,
the operations that delay a value to a later instant. Closing a cell
loop with `lift(forward, other, f)` is therefore rejected, since it
would define a value in terms of itself at the same instant, which the
semantics cannot give a meaning to; closing it with a hold whose
input snapshots the forward token is the normal case. The check runs
at close, and again at relink for cycles a `construct` creates: the
nodes a `construct` closure built are linked into the graph at commit,
and their new dependencies are checked then, at the end of the
transaction that created them, so a cycle is found the first time the
code runs ([RFD 5](./rfd-0005-transaction-protocol.md)).

The closer is consumed by `close`, so a loop cannot close twice. A
loop must close in the scope that declared it, and that is checked
against the loop node, not the closer: the build or `construct` scope
records the loops it declared and panics at scope end for any that is
still open. Moving the closer somewhere else changes nothing. In
particular, smuggling a closer into a `construct` closure through an
`Option` and taking it out on some later event compiles, and the
declaring scope still panics at its end because the loop is open when
the scope closes.

Sampling a loop cell before it is closed is a build-time panic, since
there is no value to return. Sodium's `Lazy` family is out of the
first version; a `Lazy<A>` for the initial-value case can be added
later, forced at the end of the scope after every loop has closed,
which is exactly what Sodium's `sampleLazy` does.

We tried closure-scoped loops first, `b.cell_loop(init, |b, c| ...)`,
which cannot be left open but nest one level per loop. Three entwined
loops in a protocol state machine, a block number, a retry counter and
a terminal error that each read themselves and each other, nest three
deep with the whole body in the innermost closure. That is the normal
shape of a state machine, not a corner case. A single declaration at
the top of the build, forward tokens as a parameter and definitions as
a second return value, cannot express a loop inside a constructed
screen. The flat form is what an application framework needs as well:
declare here, hand tokens to user code, close there.

## The I/O API

`Graph` has `send`, `transaction`, `listen`, `listen_cell`,
`listen_steps`, `anchor`, `sample`, `collect_garbage`,
`set_collection_policy`, `set_collect_after_every_transaction`,
`live_nodes`, `stale_operations`, `set_waker`, `pump` and `remote`.
`anchor` keeps a node alive that I/O code wants to hold
without listening to it, and returns the `Anchor` handle that holds
it. The first name for it was `Pin`, an unrelated concept in
`std::pin`; the second was `Root`, which collided with the concept an
anchor is one kind of.
`collect_garbage` runs a collection now, for the manual policy;
`collect` was the obvious name and is exactly the name the table below
retires because it means something else to a Rust reader.
`graph.transaction(|tx| ...)` hands out a `Transaction` that has only
`send`, so simultaneous inputs are one closure. `Transaction` exists
only on the I/O side, which removes the dual use this RFD complains
about. `set_waker`, `pump` and `remote` are the threaded and async
edge ([RFD 6](./rfd-0006-io-edge.md)); the collection policy, the
stress setting and the two counters are
[RFD 3](./rfd-0003-memory-model.md)'s.

Every operation that can fail has a `try_` sibling returning a
`Result`, and each family of operations with the same failure modes
has its own error enum, with no variant an operation cannot return.
The panicking variants panic on misuse, except for the operations on
collected nodes whose effect the semantics cannot observe, which are a
debug-mode panic and a release-mode no-op.
[RFD 5](./rfd-0005-transaction-protocol.md) lists the families.

We considered a scoped `rt.with_io(|ctx, ...| ...)` context with
pulled outputs. The scope adds nothing structural, since graph code
already cannot reach `Graph` during a transaction, and an application
framework needs something that owns the graph across event-loop
iterations.

## Reading a Cell

`c.sample(b)` in graph code and `graph.sample(c)` in I/O both take the
context by shared reference and return `&A` borrowed from it. Shared
borrows compose, so `format!("{} {}", c.sample(b), d.sample(b))` and
binding two samples to variables both compile; a caller that wants to
keep a value clones it, and the clone lands where the user decides to
keep the value, which is the whole philosophy of
[RFD 4](./rfd-0004-value-model.md). There is no `Clone` bound and no
closure-taking variant. Reading a read-through cell may compute and
memoize, so its memo is a `OnceCell`, which hands out a reference from
a shared borrow and is cleared at commit through `&mut`
([RFD 4](./rfd-0004-value-model.md)); taking the context by `&mut` was
the first draft, and a stub of the API showed that two samples in one
expression are then a borrow error, which makes `sample` unusable for
its main job.

## Names

Sodium's names are kept where they are good, and the book stays the
manual for those. These change, following the naming policy in
[RFD 1](./rfd-0001-guiding-principles.md):

| Sodium | Bough | Why |
|---|---|---|
| `switchS`, `switchC` | `switch_stream`, `switch_cell` | The suffix letters mean nothing to someone who has not read the book. |
| `updates` | `steps` on `Cell`, `listen_steps` on `Graph` | An operational primitive: the name says what it exposes, a cell's steps, and it carries the book's warning. Fires on every step, including a step to an equal value. |
| `value` | `steps_with_current` on `Cell`, `listen_cell` on `Graph` | Long on purpose: it fires once at creation with the current value and then on every step; the listener fires once at registration. |
| `collect` | `scan` | `collect` means something else to every Rust reader, and `scan` is the nearest Rust name; [RFD 4](./rfd-0004-value-model.md) gives the signature, which is not `Iterator::scan`'s. |
| `filterOptional` | `filter_map(\|o\| o)` | `filter_map` takes a function and covers the map-then-filter pair; the identity closure moves the `Option` through by value, so this needs no `Clone`. |
| `apply` on cells | `lift` with `\|f, a\| f(a)` | The oracle's `Apply` applies a cell of functions to a cell. Cell values are read by reference, so the cell holds `Fn(&A) -> B` and `(cf, ca).lift(b, \|f, a\| f(a))` is it; the oracle tests exercise it that way. |
| `lift` (arities 2 to 6) | `lift` on a tuple of cells, arities 2 to 6 | Rust has no variadics, and a tuple of cells is the one form: `(a, x, y).lift(b, \|a, x, y\| ...)`. Chaining binary lifts composes functions rather than lifting three cells ([RFD 4](./rfd-0004-value-model.md)). |
| `accum` | `accumulate` | The naming policy; `accumulate_mut` is the in-place form ([RFD 4](./rfd-0004-value-model.md)). |
| `map` on a cell | `map_cell` | Distinct from `map` on a stream, and spelled out. |
| `listen` on a cell | `listen_cell` | Cells are read by reference, so the listener signature differs from the stream one. |
| `StreamSink`, `CellSink` | `input`, `input_cell` | Inputs are created from `Build` and drive a stream or a cell. |
| `StreamLoop`, `CellLoop` | `cell_loop`, `stream_loop` with `close` | Same concept, flat declare and close. |
| `Operational` | gone | `defer` and `split` are ordinary methods on `Source`; `updates` and `value` are `steps` and `steps_with_current` on `Cell`, carrying the book's warning, and listeners on `Graph`. |
| `execute` (the semantics) | `construct` | What it does, in Rust words. |

Kept as is: `never`, `constant`, `map`, `map_to`, `filter`, `merge`,
`or_else`, `snapshot`, `hold`, `gate`, `once`, `sample`, `split`,
`defer`, `listen`.

New, with no Sodium counterpart: `share` and `Shared`, `node` and
`Node`, `Source` and `Event`, `Trace` and `Leaf`, `depends`,
`input_coalescing`, `input_cell_coalescing`, `anchor` and `Anchor`,
`Listener`, `keep`, `collect_garbage`, `set_waker`, `pump`, `remote`
and `Remote`, `InputSlot` and `connect`, `Build`, `Graph`,
`Transaction`, `Local` and `Threaded`.
