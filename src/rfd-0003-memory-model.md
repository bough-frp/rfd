# Memory Model

**State:** discussion · **Authors:** Zefira Shannon, Claude

Sodium's Java, C# and TypeScript implementations lean on a host garbage
collector. `sodium-rust` built its own: reference counts on every node
plus a tracing pass that needs every closure to declare the Sodium
objects it captures. That declaration tax is paid on every closure,
and a cycle through values, a cell whose value holds a token upstream
of itself, still leaks. Bough's requirement is stricter on both
counts: a leak is unacceptable, and whatever the user has to tell us
must be checked.

## Nodes Live in an Arena, Tokens Name Them

Every node lives in one arena owned by the `Graph`, indexed by `u32`.
There is no `Rc` in the graph and no `RefCell` in the node graph.
Interior mutability appears in three places, each with one job: the
memo of a read-through cell, a `OnceCell` that hands out a reference
from a shared borrow and is cleared only through `&mut` at commit
([RFD 4](./rfd-0004-value-model.md)); the flag a handle shares with
its node, so that dropping the handle needs no graph access; and the
inbox behind `Remote` ([RFD 6](./rfd-0006-io-edge.md)). A token is an
index, a generation and a graph id. A freed slot bumps its generation,
so a stale token fails the check on its next use instead of addressing
a recycled node. A slot whose generation reaches its maximum is retired
and never reused, one compare on free, so a stale token from before a
wrap can never validate; with a first-in first-out free list the churn
spreads across every slot, and retirement is astronomically rare
([RFD 7](./rfd-0007-targets.md)). This is what makes a wrong `Trace`
implementation a loud error rather than memory unsafety, and it is why
`Trace` is a safe trait.

`Cell<A>`, `Input<A>` and `Shared<A>` are `Copy`. `Stream<A>` is
move-only because it is linear ([RFD 4](./rfd-0004-value-model.md));
it is still the same twelve bytes. The phantom in every token is
`PhantomData<fn() -> A>`, so a token is `Send` whatever `A` is, which
[RFD 6](./rfd-0006-io-edge.md) relies on.

The arena and every per-transaction structure sit behind a `pub(crate)`
storage seam from the first increment, and the engine allocates only at
build and inside `construct`: mark and order buffers are reused,
dispatch lists are reused, and a child transaction pulls from the
split's iterator rather than queueing items. A graph that is not
growing does not allocate. That is what lets the same engine run on a
microcontroller with an allocator it never calls after start, and what
lets a bounded storage backend with a fixed number of slots land later
behind the seam without touching the protocol; `construct` can then run
out of slots, which is a panic that poisons, an `Exhausted` variant on
three error enums, and a major version
([RFD 7](./rfd-0007-targets.md)).

## Liveness Is Reachability From Explicit Roots

A node is alive when it is reachable from a root. There are exactly
three kinds of root:

1. whatever the build closure returned, which is why that type must
   implement `Trace`;
2. every live listener;
3. every live `Anchor` handle, which I/O code takes with
   `graph.anchor(&token)` on a token it wants to hold without listening
   to it.

A node's reach, what it keeps alive, is its dependencies, the tokens a
`Trace` walk finds inside a stateful cell's committed value, a switch's
currently selected inner (which is such a value), and the explicit
`depends` declarations below. Reach is wider than dependency: the
marking walk of a transaction follows dependents lists only, so a
`depends` declaration never orders evaluation and can never read as a
cycle ([RFD 5](./rfd-0005-transaction-protocol.md)).

Collection is mark from the roots, sweep the arena, bump the
generation of every freed slot, and prune dead nodes out of their
inputs' dependents lists. Nothing is counted, so a cycle through
values is collected like anything else. A node reachable only from a
token I/O code holds but neither anchored nor listened to is
collected, and the next use of that token is an error: anchor it or lose
it. This is semantically right. A deselected inner cell that something
still names keeps accumulating, because it is reachable; one that
nothing names can never be observed again.

Why not reference counts. Counted tokens would root what I/O code
holds and what values hold, but counts cannot see cycles; `sodium-rust`'s
answer was declared dependencies per closure, and it still leaks
through user structs. Weak references, the C++ implementation's
answer, collect nodes that must keep accumulating while deselected.
Tracing from roots is the only model in which "unreachable" means
"unobservable forever", and it needs no counts, so tokens can be
`Copy`.

## What the User Declares

Two things, both checked.

Every type held in a cell implements `Trace`, derived with
`#[derive(Trace)]` next to the `Clone` most such types already derive,
with `#[trace(skip)]` for fields that cannot hold tokens. A value of a
foreign type that holds no tokens goes in a `Leaf<T>` wrapper, which
traces nothing and derefs to `T`; the orphan rules forbid implementing
`Trace` for another crate's type, so a macro declaring one a leaf could
only ever run inside this crate. Implementations ship for the
standard library's types; those for `HashMap`, `HashSet` and `Instant`
need `std`, and the one for `Arc` needs pointer atomics. The bound sits
on `hold` and the other operations that persist a value, so a stream
of non-`Trace` values can be mapped, filtered and merged freely and
fails only where it would persist. Stream event types need nothing
beyond what their adapters need, because slots are cleared before a
collection and only holds persist.

`Trace` is a safe trait. The obligation is real, but the consequence
of getting it wrong is a stale-token error, and `unsafe` should mean
memory safety and nothing else. `Ord` and `Hash` carry
compiler-unverifiable invariants and are safe traits for the same
reason. The `gc` crate marks its `Trace` unsafe because its collector
frees memory on the strength of it; ours does not, no memory is ever
touched through a stale token. Making hand-written implementations
`unsafe` would also lock a crate with `forbid(unsafe_code)` out of
writing one.

A closure that captures a token that is not upstream of its own node
declares it: `b.depends(&node, &[&clicks, &total])` states that a node
keeps those tokens alive, one declaration per closure, callable after any
construction, taking references so it does not consume a linear
stream; the slice is heterogeneous, since the tokens a closure captures
are rarely all of one type. Upstream captures need no
declaration, since reachability already covers them. The declaration
is needed exactly for backward and unrelated references, which are the
ones that create cycles, and the tracing walk then handles the cycle.
We considered a deps-list parameter on every closure-taking
constructor, `sodium-rust`'s shape, which doubles that part of the
API, and a deps parameter on `construct` alone, which misses a `map`
that emits a captured token after the node it came from has become
unreachable.

An undeclared capture cannot be made a compile error without making
tokens unusable as data, and we want the data. Its failure mode is a
stale token: a loud error at the point of use, never a leak and never
a read of a recycled node. A collection-stress setting on the graph,
`set_collect_after_every_transaction`, makes that error deterministic
in tests, and the public live-node count, `live_nodes`, is how the
no-leak requirement is asserted: build, drive, drop, collect, compare.

## When Collection Runs

Never inside a transaction. Automatic by default and amortized: after
a transaction, collect when the nodes allocated plus the roots
released since the last collection exceed the live count. Garbage is
made by unrooting as much as by allocating, and a graph built once
that then drops listeners allocates nothing, so a trigger on
allocation alone would never fire for it; a graph that neither
allocates nor drops a root never pays. A manual policy is available,
set with `set_collection_policy` and run with `collect_garbage`, for a
frame loop that wants to collect at frame end or a high-rate loop that
wants to choose when it pays. Manual-only was rejected because UI code
should never think about it; automatic-only because the shallow
high-rate shape wants to choose.

## Handles

A handle borrows nothing from the graph, so the graph can be driven
while handles are held. `Listener` and `Anchor` are RAII: dropping the
handle unlistens or unanchors by flipping a flag the node shares, so
dropping one inside a listener callback needs no graph access. A
handle is live until it is dropped, and a live handle is a root.
`keep()` consumes the handle and leaves its listener or anchor live
for the graph's lifetime, a root without a struct to hold it, because
every real application has process-lifetime listeners and a field
called `_listeners` that exists to be ignored is the alternative.
`unlisten()` exists for symmetry with Sodium. A linear stream is
consumed by `listen`, so once that listener is dropped the stream can
never be observed again and is collected unless something else roots
it. Rust-level reference cycles between a listener closure and the I/O
state that owns its handle are the user's problem and get a
documentation note, not machinery.

## Operations on Collected Nodes

Sending to a collected input, listening to a collected stream, or
anchoring a collected node has no effect the semantics can observe. In
the panicking variants these are a debug-mode panic and a release-mode
no-op, counted on the graph so a release build can still report that
it is dropping sends; the `try_` variants return `Err(Stale)` in both
modes ([RFD 5](./rfd-0005-transaction-protocol.md)). Sampling a
collected cell must return something, so it always fails.
