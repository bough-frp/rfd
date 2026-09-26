# Targets: No-std Tiers, Input Slots, and Host Embedding

**State:** discussion · **Authors:** Zefira Shannon, Claude

Three targets became first class after the first six RFDs were written:
bare metal, meaning Cortex-M3 and M4 boards and later a Cortex-M0; the
Bevy game engine as a host; and the web through `wasm32-unknown-unknown`
and wasm-bindgen. Each was checked against the design as it stood, with
the research in [`research/`](./research/2026-09-22-no-std-handoff.md)
carrying the evidence, and each changed something. This RFD records the
decisions that are costly to undo: the tiers the engine ships in, the
input slot, what a target without atomics keeps, and the rules a host
must follow. The smaller consequences are edited into the RFDs they
belong to.

## Two Tiers, and a Third Later

The core is `no_std` over `alloc`, and `std` is a feature, on by
default, that adds the thread-id guard on `Remote`, `Trace` for the
standard collections and `Instant`, and the standard mutex under input
slots. The memory-model principle that a graph pays for allocation only
while it grows ([RFD 3](./rfd-0003-memory-model.md)) becomes a rule: the
engine allocates at build and inside `construct`, and nowhere else. Every
per-transaction structure is reused, and the arena and those structures
sit behind a `pub(crate)` storage seam from the first increment. Both
boards we own have allocators, so this tier runs the button-to-LED
milestone on the F303 Discovery and on the Due.

A third tier follows once the engine works: a bounded storage backend
behind that seam, with a fixed number of slots the caller provides, so a
graph that never grows never allocates at all. It brings one new failure,
`construct` finding no free slot, which is a panic that poisons the
graph and an `Exhausted` variant on the three error enums whose
operations run transactions. Under the rule that no enum carries a
variant an operation cannot return, those variants do not exist until
then, and adding them is a breaking change; the bounded tier is a major
version, and we accept that rather than mark every enum
`#[non_exhaustive]` today or route exhaustion through a second call.
Exhaustion is not a semantics deviation: a program that exhausts the
arena gets no answer, loudly, the way a `std` program aborts on
allocation failure, never a different answer.

A static engine, the graph fixed at compile time in the types with no
node table and no allocation, was explored and rejected for the core.
The exploration in
[`research/`](./research/2026-09-23-static-engine-exploration.md)
compiled the alternatives for all three Cortex-M targets. The static
engine is smaller by a wide margin, about nine kilobytes of code and 1.4
kilobytes of RAM for a three-hundred-node program against seventy-six
and 7.6 for the bounded dynamic engine, and it pays for that with a
second front end, since a hand-written type-level builder stops being
checkable somewhere between a hundred and three hundred nodes and a
proc-macro front end sees syntax rather than types; a second testing
harness, because a type-level program cannot be generated at runtime for
the oracle; and either a subset of the semantics without `construct` or
pooled `construct` that brings a collector back per pool. It is a
separate product with the same semantics, worth building someday as its
own project, and not this one.

The dependency policy in [RFD 1](./rfd-0001-guiding-principles.md)
changes by one clause: the engine has zero runtime dependencies in the
default feature set. A `critical-section` feature adds the crate of that
name, which is the seam every embedded HAL implements, to guard input
slots on bare metal. A `Lock` trait the embedder implements would have
kept the count at zero and made every embedded user write glue the
ecosystem already wrote once; in-tree inline assembly per architecture
would have been a port of the same crate.

## What a Target Has

Everything the I/O surface shares between threads is an `Arc`, and an
`Arc` needs compare-and-swap. A Cortex-M3 or M4 has it; a Cortex-M0 has
32-bit loads and stores and no read-modify-write, and `alloc::sync` does
not exist there. So `Threaded`, `Remote`, the unit queue and their errors
exist only where the target has pointer atomics, the way `std::sync::Arc`
itself does, and a Cortex-M0 gets `Local` mode and input slots, which is
what an interrupt-driven main loop needs. `core::task::Waker` is in
`core` and needs no atomics, so `set_waker` exists everywhere. The M0 is
not a board we own; it is the floor the design reaches without a
`portable-atomic` dependency, verified by compiling the memo, the waker
and a slot for `thumbv6m-none-eabi`.

On `wasm32-unknown-unknown` the gate is true, so `Threaded`, `Remote` and
the queue exist with no threads to use them, and `Local` is the web mode:
a stable build cannot share a graph across web workers, and a worker
feeds the graph by posting a message to the main thread. On a
non-atomics wasm build `JsValue` and every `web_sys` handle are `Send`,
so `Threaded` would accept them, while a `Closure` is not; nothing to
decide, one sentence in the mode documentation.

`Listener` and `Anchor` carry the mode, `Listener<M = Local>`, because
the flag a handle shares with its node is a counted cell in `Local` and
an atomic in `Threaded`, and the parameter is defaulted so `Local` code
never writes it. Type-erasing the flag behind a virtual call would hide
a real difference for nothing.

## Input Slots

Interrupt handlers, GTK callbacks and tokio tasks all hand input to a
single-threaded core without racing a transaction. The queue of units
behind `Remote` serves the last two. An interrupt handler needs something
else: it has no allocator, no lifetime and no `Arc`, it fires in bursts,
and a queue that grows during a burst is a fault on a microcontroller.

An input slot is a value the writer owns, which for an interrupt handler
means a `static`:

```rust
static PRESSES: InputSlot<u32> = InputSlot::new(|a, b| a + b);
```

It holds one pending event. A write when one is pending folds the two
with the slot's fold, pending on the left, so a burst between two pumps
becomes one event and the slot never grows. The fold is a `fn` pointer so
that the type can be named in a `static`; a closure that captures nothing
converts to one. Every slot has a fold, and dropping the older event is
spelled `keep_latest`, so the drop is declared rather than silent, the
way the first design refused silent last-wins for a double send. An
overflow counter was the alternative, and a counter is a decision nobody
made.

Build connects a slot to an input, `b.connect(input, &PRESSES)`, callable
more than once for one input with one slot per producer; a second
producer gets a second slot. The slot's fold and the input's coalescing
function are independent declarations with different jobs: the fold
combines a burst between two pumps, the coalescing function combines two
sends inside one transaction, and slots never cause the second, since
`pump` runs each pending slot as a transaction of its own in connection
order. A constructor per combination would have doubled the four input
constructors to six.

Two slots are never simultaneous, and that is a rule about the semantics
rather than a convenience. Simultaneity means caused by the same external
event, and `merge` combines simultaneous events through its function, so
two unrelated inputs made simultaneous by the timing of a drain would be
combined into one event, a different answer rather than a coarser one.
The Bevy port's research reached the same conclusion for frame batching
([`research/`](./research/2026-09-23-frp-host-embedding-requirements.md)).
A genuinely simultaneous multi-input event is a tuple input, or a queued
unit. The per-transaction cost is a counter and reused buffers, so five
sensor slots at a kilohertz are five transactions a millisecond.

The law the fold obeys: a slot's external sequence is cut into runs by
when the driver pumps, which is timing the semantics do not see, and each
run folds left to right in arrival order into one event. So the fold must
be associative, and it need not be commutative because a slot has one
producer. That is documented and property-tested against the oracle: for
random event sequences and random partitions into runs, the engine's
output equals the oracle fed the folded runs. Requiring commutativity
would have allowed several producers on one slot and forbidden
`keep_latest`, the commonest fold.

The slot is guarded by a critical section on bare metal, through the
`critical-section` feature, and by the standard mutex under `std`. A web
build keeps `std`; the crate ships no critical section for wasm, so a
`no_std` web build registers a single-core no-op itself. The waker a
slot write wakes is a `core::task::Waker`, so an embedded async executor
that builds wakers without an allocator drives the graph the same way a
desktop runtime does. The first sketch had a per-input slot inside the
graph's own inbox; that puts the slot behind the graph's ownership, and
reaching it from an interrupt needs an `Arc` the M0 lacks or a `&'static`
an owned graph cannot give without a leak.

## The Guard, and Retirement

`Remote::send` from the driver's thread during a transaction is an error,
and the check needs a thread id, which `std` has and bare metal does not.
The guard exists under `std` only. On bare metal the legitimate other
context is an interrupt handler, which this RFD serves with slots, so a
`Remote::send` from one is documented misuse; a Cortex-M port that reads
the IPSR register to tell a handler from the main thread is a later
addition if the misuse bites, not a maintenance surface for a
hypothetical.

A slot's generation is a `u32` bumped on every free. A device that runs
for months could wrap it, and a stale token from before the wrap would
then validate. A slot whose generation reaches its maximum is retired
and never reused, one compare on free; with a first-in first-out free
list the churn spreads across every slot, so retirement is astronomically
rare in practice and the stale-token guarantee stays true without a
footnote. Widening to `u64` would make a token sixteen bytes and cost an
atomic the M3 and M4 cannot update. A harness that estimates a program's
uptime from its measured churn is an idea in
[`notes/`](./notes/2026-09-23-slot-churn-simulation.md).

## Hosts

A host is a runtime that owns the schedule: a game engine's frame loop,
a browser's event loop. Bough's answer to both is the same shape. The
graph is library-owned; a driver the host schedules pumps a `Remote`-fed
inbox inside the host's own critical section; each external cause is one
unit and one transaction; latency is the distance from a send to the
next pump, a property of where the embedder placed the driver, which an
adapter states.

Bevy gets a `bough-bevy` adapter after the performance bar is measured,
on the same terms as `bough-tokio`: the graph a resource, an exclusive
system as the runner, every host message its own unit, never a frame
batched into one transaction. The health-and-shield slice from the
Bevy port's research is the adapter's example, and its semantics are
tested in the oracle long before the adapter exists. A fully ECS-native
port, nodes as entities and edges as the host's relationships, stays a
separate project; it is that project's experiment, and the fan-in wall it
hit is a property of that bet. The research also asked for the graph
representation to be a public backend interface, so an embedder could
choose library-owned or host-owned storage. Rejected: the trait would be
the whole of [RFD 5](./rfd-0005-transaction-protocol.md) guessed before
any of it exists, and chain fusion is only possible when the library
owns storage. The internal storage seam is the narrower thing both new
targets need, and a public trait, if ever, is extracted from a working
engine.

The web fits closely, and the research in
[`research/`](./research/2026-09-23-wasm-target-research.md) verified
each claim by compiling or running it. `Local` is the web mode. A DOM
callback sends through `Remote`, never directly, because `dispatchEvent`
runs listeners synchronously and nested: a closure holding the graph in
`Rc<RefCell<_>>` that sends directly traps on a double borrow the moment
a Bough listener dispatches a DOM event whose closure sends again, and on
that target a trap leaves the module callable but damaged. The driver is
a future spawned with `spawn_local` that stores its waker, pumps and
returns pending; the pump runs as a microtask, before the next task and
before the next paint. `bough-web`, after the bar like the others,
exposes the graph only to the driver and routes every DOM closure through
the inbox; direct sends stay possible with the hazard stated. The trap
to avoid is pumping on `requestAnimationFrame`, which delays every
transaction to the next frame and stops in a hidden tab.

That target has no unwinding: a panic is a trap, no drop guard runs, and
`catch_unwind` does not exist. This is why poisoning became the
transaction-in-progress flag itself rather than a bit a guard sets on
the way out ([RFD 5](./rfd-0005-transaction-protocol.md)), and why a
web adapter tears the graph down at the first `Poisoned` its driver
sees.

## Testing and CI

Continuous integration checks the core for `thumbv7m-none-eabi`,
`thumbv7em-none-eabihf` and `thumbv6m-none-eabi` without `std`, for
`wasm32-unknown-unknown` with and without it, and on the minimum
supported Rust version, so a standard-library use that creeps into the
core fails the build the day it lands. Once the engine exists, a
`wasm32-wasip1` leg under wasmtime runs the engine and the oracle on a
32-bit abort target, with proptest's default features off, which do not
build for wasm, and the build-time panic tests explicitly ignored there,
where libtest would otherwise skip them in silence. The Node leg under
`wasm-bindgen-test` is for `bough-web` alone. Wasm code size is reported
as information beside trivial-payload overhead, about two hundred and
seventy bytes per monomorphized node type after `wasm-opt`; there is no
wasm analogue of the instruction-count gate.

The first embedded milestone is a button press through a small graph to
an LED on the F303 Discovery, bare metal. It lives under `examples/` in
the `bough` repository as a crate of its own, cross-compiled in
continuous integration for its target and never run there.
