# Glossary

Bough is a functional reactive programming library that implements the
Sodium denotational semantics with an API that cleanly separates
building FRP logic from driving it with I/O. This glossary is the
project's language; the design behind each term is in the RFDs.

## Language

### Time

**Instant**:
A point in the semantics' time. Everything within one instant is
simultaneous, and a stream has at most one event per instant.
_Avoid_: tick, time step, frame

**Transaction**:
One instant as the engine runs it, from the sends that open it to the
listeners that close it. The initial build is transaction zero.
_Avoid_: batch, tick

**Child transaction**:
A transaction that `split` or `defer` schedules to run after its parent and
before the next external instant.
_Avoid_: post-transaction, sub-transaction

**Event**:
A source's value at one instant. A stream *fires* events. A toolkit's event
becomes a Bough event when I/O code sends it into an input; "event loop"
and "event-driven" keep their ordinary meanings.
_Avoid_: occurrence (the semantics' `occs`, the same thing), firing (as a noun), message

**Step**:
A change of a cell's value from one instant to the next, including a change
to an equal value.
_Avoid_: update, change

### Streams and cells

**Stream**:
A stream with exactly one consumer, whose events move through by value.
Every constructor takes it by value, so using it twice is a compile error.
_Avoid_: event stream, linear token

**Shared stream**:
A stream with any number of consumers, each of which clones the event. The
only way to give a stream more than one consumer.
_Avoid_: broadcast stream, fan-out stream

**Linear**:
Having exactly one consumer. Streams and chains are linear.
_Avoid_: affine, single-use, unique

**Fan-out**:
Giving a stream more than one consumer, which is always explicit, through
`share`.
_Avoid_: splitting (that is `split`), broadcasting

**Cell**:
A value that exists at every instant. Cell values are read by reference;
the engine clones one only for `steps` and `steps_with_current`, which
need a value of their own.
_Avoid_: behavior, signal, property, variable

**Hold**:
A cell that keeps the latest event of a stream, starting from an initial
value. Holds, accumulators and constants are the stateful cells; a
constant is a hold that never steps.
_Avoid_: register, latch, state cell

**Accumulator**:
A cell whose value is folded from a stream's events, either by returning a
new state or by mutating the state in place.
_Avoid_: reducer, fold cell

**Read-through cell**:
A cell computed from other cells when it is read and memoized until one
of them steps: `map_cell`, `lift`, `switch_cell`. It steps when an input
steps, so marking reaches it, but its function runs only on read. The
other kind of cell is stateful, a hold, an accumulator or a constant,
and there is no third kind.
_Avoid_: derived cell, lazy cell, computed cell

**Input**:
A stream or cell driven from outside the graph, and the token I/O code sends
with.
_Avoid_: sink, source, port, event sink

**Source**:
The role of anything that yields events: a stream, a shared stream, or a
chain. An input is a source; a source is not necessarily an input.
_Avoid_: producer, emitter

### Building

**Token**:
The name of a node that the four token types carry: `Stream`, `Shared`,
`Cell`, `Input`. A token has no method that creates a node without a build
context.
_Avoid_: handle, reference, id

**Handle**:
An RAII object whose drop has an effect: `Listener` and `Anchor`. A handle
borrows nothing from the graph, and carries the graph's mode.
_Avoid_: token, guard, subscription

**Node**:
Anything in the graph with an identity of its own, created by a
materializer. `Node` is also the bound `listen` takes, which only
`Stream` and `Shared` satisfy: a chain is not a node.
_Avoid_: vertex, operator

**Adapter**:
An operation that transforms a source's events without creating a node, and
the type it returns: `map` is an adapter, `Map<S, F>` is its type.
_Avoid_: stage, transformer, combinator (for these)

**Chain**:
A linear sequence of adapters with no materializer. The first materializer
consumes it, and its adapters fuse into that node.
_Avoid_: pipeline, builder, lazy stream

**Materializer**:
An operation that creates one node, from a chain or from a cell, and takes
the build context to do it.
_Avoid_: terminal operation, consumer, sink

**Dependency**:
What a node is marked from: a stream node's inputs, and the cells a
read-through cell is computed from, so that a step in one reaches the
other. A cell read inside a stream function is not a dependency, because
a cell is read as it was before the instant.
_Avoid_: edge, link, upstream (as a noun), reach (that is for collection)

**Build context**:
The context every node-creating operation requires. It exists inside the
build closure and inside construct closures, and nowhere else, and it
carries the graph's mode.
_Avoid_: builder, transaction (Sodium's word for it)

**Graph code**:
Code that runs with a build context or as a node function: the build
closure, construct closures, and the functions given to adapters and
materializers.
_Avoid_: FRP code, logic, reactive code

**Construct**:
Creating nodes during a transaction, from a construct closure. The only way
logic is added after build.
_Avoid_: dynamic construction, runtime wiring, late binding

**Scope**:
The extent of one build context: the initial build, or one run of a
construct closure. A loop closes in the scope that declared it.
_Avoid_: session, phase

**Loop**:
A cycle in the graph, declared with a forward token and closed later with a
definition. Every path around a loop passes through a hold, an accumulator,
a `split` or a `defer`.
_Avoid_: cycle (for the construct; a cycle is what a loop makes legal), recursion

**Forward token**:
The token a loop hands out before its definition exists.
_Avoid_: placeholder, forward declaration

**Closer**:
The value that defines a loop, consumed by `close`.

### Driving

**I/O code**:
Code that holds a `Graph` or a `Remote`, listeners included.
_Avoid_: the I/O world (Sodium's phrase, kept only when quoting it), the outside, the shell

**Edge**:
The I/O boundary: the tokens I/O code holds, which are whatever the build
closure returned and whatever flowed out as data since.
_Avoid_: boundary, surface, ports; never a graph edge, which is a dependency

**Driver**:
Whoever owns a graph and runs its transactions: a thread, a future, a
host's system, or a bare-metal main loop.
_Avoid_: runtime, executor, owner

**Listener**:
An I/O callback attached to a node, run after commit with no graph access,
and the handle that keeps it attached. `listen_cell` and `listen_steps` are
the I/O forms of `steps_with_current` and `steps`, Sodium's `value` and
`updates`.
_Avoid_: observer, subscriber, callback (for the attachment)

**Anchor**:
The handle that keeps a node alive from I/O code without listening to it,
taken with `anchor`. One of the three kinds of root.
_Avoid_: pin, root (that is the concept)

**Remote**:
A `Send + Clone` endpoint for sending into a graph from any thread. A
remote send or a remote transaction is queued as one unit, and the driver
runs each unit as one transaction when it pumps. Exists where the target
has pointer atomics.
_Avoid_: sender, proxy, channel, handle (its drop does nothing)

**Unit**:
What the inbox queues: one remote send, or the several sends of one
remote transaction, run by the driver as one transaction. A unit is the
declaration of one external cause, never split and never merged.
_Avoid_: batch, message, job

**Pump**:
Running every pending input slot, each as one transaction in connection
order, then every queued unit, each as one transaction in arrival order.
_Avoid_: poll, drain, flush

**Input slot**:
A static mailbox for one input, placed by the code that writes it, folded
in place, and drained by the driver as one transaction per pending slot.
Connected to an input at build; one slot per producer.
_Avoid_: mailbox, buffer, interrupt queue

**Fold**:
A slot's function for combining a pending event with a new one, pending on
the left, associative. Not the input's coalescing function, which combines
two sends inside one transaction.
_Avoid_: coalescer (that is the input's), reducer, accumulator (that is a cell)

**Mode**:
Whether a graph is `Local` or `Threaded`: whether what it stores must be
`Send`, and whether the graph itself is. Handles carry it.
_Avoid_: flavor, threading model

**Tier**:
One of the engine's feature levels: the `no_std` core over `alloc`, and
`std`; a bounded storage backend is a later tier.
_Avoid_: profile, mode (that is `Local` or `Threaded`), edition

### Memory

**Root**:
Something that keeps a node alive: the value the build closure returned, a
live listener, or a live anchor. A handle is live until it is dropped, and
`keep` makes it live for the graph's lifetime.
_Avoid_: anchor (that is one kind), pin, owner

**Reach**:
What a node keeps alive: its dependencies, the tokens `Trace` finds in a
stateful cell's committed value, and its `depends` declarations. Reach is
wider than dependency: a `depends` declaration never orders evaluation
and can never read as a cycle.
_Avoid_: reference (a Rust word), liveness edge, retention

**Stale**:
Of a token: its node has been collected.
_Avoid_: dangling, dead, expired

**Foreign**:
Of a token: it belongs to another graph.
_Avoid_: mismatched, alien

**Poisoned**:
Of a graph: a transaction never finished, because a panic escaped it; the
transaction-in-progress flag stays set and every later call fails, remote
sends included.
_Avoid_: broken, corrupted, tainted

**Collection**:
Reclaiming the nodes no root reaches. Never runs inside a transaction.
_Avoid_: GC in prose, sweeping, cleanup

### Testing and performance

**The semantics**:
The Sodium denotational semantics, version 1.1, as the executable Haskell in
the Sodium repository. What the engine is held to.
_Avoid_: the spec, the reference implementation

**Oracle**:
The semantics ported to Rust as lists of time-stamped values, which the
engine is property-tested against.
_Avoid_: reference model, golden model

**Shape**:
One of the three benchmark workloads, UI, frame and shallow, each with a
hand-written imperative baseline.
_Avoid_: scenario, benchmark case

**Bar**:
The performance target: within a factor of three of the baseline on
realistic per-node payloads.
_Avoid_: budget, goal, SLA
