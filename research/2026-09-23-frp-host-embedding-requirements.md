# Embedding FRP in a host runtime: requirements from a Bevy port

Research handoff for an agent designing a general-purpose FRP library. The
research is a port of [Sodium](https://github.com/SodiumFRP) to the
[Bevy](https://bevy.org) ECS, carried far enough to hit real walls and stopped
before implementation. Everything below is grounded in something observed, and
says so.

**Primary sources, all public.** Do not take this document's word for anything
it summarises:

- Decision records: <https://github.com/RadicalZephyr/bevy-sodium/tree/main/docs/decisions/records>
  — `0002` is the transaction model, `0003` is the evidence that motivated it.
- Runnable experiments: `docs/decisions/experiments/src/bin/` in the same repo.
- A correctness bug the work surfaced in an existing implementation:
  <https://github.com/RadicalZephyr/sodium-rust/issues/52>
- The specification the port is against, vendored with its own licence:
  `docs/reference/sodium/denotational-semantics.md`.

Facts about Bevy below were read from vendored sources of `bevy_ecs`,
`bevy_app` and `bevy_time` **0.19.1** in September 2026. They are third-party
capabilities and will age.

---

## The finding that should shape your design

Hosts commonly make callback order **arbitrary** and ask you not to depend on
it. Bevy's documentation says the relative order of observers watching the same
event "is considered to be arbitrary" and recommends making no assumptions about
it.

That is a request, not a guarantee. Nothing checks that a program's result is
actually independent of the order, and Bevy's schedule ambiguity detection
defaults to `LogLevel::Ignore`, so nothing warns either.

FRP makes the same order undetectable **because the semantics guarantee the
result cannot depend on it.** Those two positions sound alike and are not the
same thing. The gap between them is the entire value proposition, and it is
worth more than the "six plagues of listeners" framing that motivated the work:
the plague list is a 2015 book's problem inventory for GUI listeners, and
scoping a library to it imports someone else's problem.

We demonstrated the gap rather than asserting it. The same instant applied to
the same state produced **two different permanent final values** depending on
which callback the host happened to run first, and the derived values fired once
per mutating callback with intermediates that were not merely early but wrong.
The fix a host expert reaches for — mutate in callbacks, derive once afterwards
— removed the spurious firings and **left the order-dependence untouched.** That
last result is the clearest single thing this research produced.

---

## Requirements

### R1. The embedder defines the instant; wall-clock batching must be impossible

The tempting simplification when embedding in a frame-based host is one
transaction per tick. It is **unsound, not coarse.** Simultaneity in FRP means
*caused by the same external event*; `Merge` coalesces simultaneous events
through its combining function, so two causally unrelated inputs that happened
to land in the same tick get combined into one event, and every combinator
downstream inherits the error.

Design so that an embedder cannot accidentally express this. A time type that is
a host tick count invites it; a hierarchical, opaque instant does not.

### R2. Separate the transaction *boundary* from the transaction *content*

Atomicity generally requires exclusive access to host state, and that is
expensive: in Bevy the only construct guaranteeing no other system observes
intermediate state is an exclusive system, which is a full barrier in both
directions.

Read naively, per-send transactions mean one barrier per send, which is what
makes tick-batching attractive and drives designers into R1's trap. **Queuing
dissolves the conflict:** sends enqueue, one runner takes the host's critical
section once and drains N sends inside it, each with its own instant. N
transactions cost one barrier.

Make this the documented embedding pattern, not something each embedder
rediscovers. It is what makes semantically correct instants affordable.

### R3. Make the graph representation a backend, not a premise

The port's central bet was that the dependency graph *is* the host's data
structure — nodes are entities, edges are host relationships — rather than
something the library owns. That buys native parallelism, storage, inspection
and tooling, and costs re-deriving every semantic guarantee from scratch.

A library intended for diverse contexts should not choose once. Put the graph
behind an interface the semantics are written against, so an embedder can pick
library-owned (portable, predictable) or host-owned (integrated, inspectable)
without forking the library.

### R4. Do not assume the host's relationship primitive expresses fan-in

Bevy's relationship primitive is one-to-many: an entity points at **at most
one** target through a given relationship component. `lift`, `merge` and
`snapshot` are all n-ary. This is the first structural wall the port hit and it
is still unresolved there.

Your edge representation must support n-ary inputs natively. Treat any host
primitive that looks like a graph edge as expressing fan-*out* until proven
otherwise.

### R5. Specify the listener contract the semantics do not cover

The denotational semantics says what values a listener receives. It says nothing
about which thread runs it, whether it may re-enter, or what it may touch. Those
omissions bite every user.

Concretely: a `std::sync::Mutex` written from a listener and read from the main
thread **deadlocked** in an existing implementation; an atomic counter did not.
Root cause was not established. A user should not have to discover this.

State the thread, the reentrancy rules and the permitted operations for callback
code, as part of the library's contract rather than its folklore.

### R6. Feedback is the common case, not an advanced feature

Every correctness problem encountered involved a loop. The bug filed upstream is
exactly this shape: a cell held through a loop, whose own update stream reaches
a second cell, and lifting the two together **stopped the first cell updating at
all** — a purely observational derived value changing its own input, which
`Apply` cannot do.

The triggering shape is mundane. It is what "shields absorb damage before
health" looks like: one piece of state upstream of another's update. Anything
with feedback produces it.

Put loops, and specifically **diamonds through loops**, in the test matrix from
the first commit. Do not treat them as the hard case to get to later.

### R7. Ship the semantics, and make tests cite them by section

Vendoring the formal specification into the repository turned arguments that
would have been assertions into citations with section numbers. It also creates
the only defensible test convention for a port: a test asserting the
implementation diverges from the specification is written against the
specification, so it stays correct after the fix — unlike a test written against
internals the change is going to replace.

If your library has a denotational semantics, ship it, version it, and make the
test suite reference it. If it does not have one, that is the first thing to
write, not the last.

### R8. Make order independence checkable, not just claimed

This follows from the load-bearing finding. A library whose pitch is "the order
cannot matter" should be able to demonstrate that for a given graph, or at
minimum detect when an embedder has stepped outside the guarantee. Hosts will
not do it for you — Bevy's own detector for the analogous problem ships off.

### R9. Make the latency consequence of placement explicit

Where the runner sits in the host's cycle determines when a send becomes
visible. In the Bevy design, a send during the update phase is not evaluated
until the *next* cycle's pre-update phase — a one-frame floor, and the thing
most likely to make the library feel wrong to use.

Whatever placement an embedder picks, the resulting latency should be a stated
property, not something discovered in profiling.

### R10. Expect no abort, and say so

The host may offer no rollback. Bevy has none: a panic part-way through a
command drain leaves already-applied commands applied. If your transaction
cannot abort, state it plainly rather than letting the word "transaction" imply
a guarantee you do not provide.

---

## Opportunities an ECS host offers that others may not

**The scheduler can be fed from the dataflow graph.** Bevy supports injecting
computed dependency edges through a build-pass interface, and ships an example
of one in the engine itself. An FRP graph's topology is exactly the information
a scheduler wants, and this is the most interesting unexplored direction the
research found. Caveat: a schedule rebuild recomputes a conflict matrix over
every pair of systems, so it is not free.

**The host's buffered-message mechanism may be a better transport than its
callback mechanism.** Bevy has both, with close to opposite characteristics —
the buffered one has no listener registration at all, is parallel-safe on the
read side, and is ordered by the schedule. Distinguish what your library needs
from a *transport* from what it needs from a *scheduler*; they may be satisfied
by different host facilities.

**Cells may be the largest thing you bring.** There is no behaviour/cell
analogue anywhere in `bevy_ecs` 0.19.1 — no current-value concept, no
compositional derived state. Change detection is query-based. `hold` and `lift`
are not conveniences in this setting, they are the missing abstraction.

**Nested host operations may already match hierarchical time.** Bevy's command
queue drains depth-first across generations: work queued by an operation runs
after it and before the next sibling. That is the same ordering as `Split`'s
child time steps. Whether the correspondence can carry weight is untested, but
it is the kind of structural match worth looking for in any host.

---

## What this research did **not** establish

Be careful quoting it beyond these limits.

- **No working FRP arm.** The comparison's FRP side was blocked by the upstream
  bug. The defect in the host's model is demonstrated; FRP *delivering* order
  independence in practice is not.
- **No performance data at all.** Deliberately out of scope throughout. The cost
  of the added barrier and the one-frame latency floor are consequences to be
  measured, not premises.
- **No implementation.** The port is decisions and evidence; the library is
  eleven lines of sketch.
- **A comparison biased toward code shape.** The FRP arm used a non-ECS-native
  implementation, so nothing about ECS-native ergonomics was tested.

## Method note, if you run your own comparison

Auditing a host against a list of known problems could not fail — the host's own
documentation conceded two of them in writing, so the exercise produced
citations rather than a decision. What produced a decision was **building the
same feature twice with a requirement change halfway**, on the smallest domain
containing a *diamond*: a value derived from two inputs that can both change in
the same instant. Pick the domain for that property; a fold over a sequence, or
per-item state that never interacts, exercises nothing FRP offers.

And steelman the incumbent. The most useful result here came from implementing
the fix a host expert would reach for and showing precisely how far it gets.

## Suggested skills

Call these with the Skill tool. The available set varies between sessions —
check what is actually listed before assuming.

- **`sodium-frp`** — Sodium's primitives and the failure modes of reactive code
  (loops that deadlock, first events missed, duplicate firings). Directly
  relevant to R1, R5 and R6.
- **`engineering:system-design`** and **`engineering:architecture`** — for the
  backend-boundary question in R3 and for recording the resulting decisions.
- **`anthropic-skills:domain-modeling`** — terminology and ADRs; useful because
  the vocabulary here (instant, transaction, cell, node) is where the design
  arguments actually happen.
- **`grilling`** — several of these requirements have multiple workable answers
  and no obvious winner. Stress-testing before committing is worth a session.
- **`tdd`** — pairs with R6 and R7: the loop and diamond cases want to exist as
  failing tests before any graph does.
