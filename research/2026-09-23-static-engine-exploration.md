# A no-allocator core for Bough: static engine, bounded dynamic engine, or neither

_2026-09-23. An exploration for the Bough design session, answering
`HANDOFF-static-engine.md` and its addendum. Transient working state, not a
record: nothing here has authority over the RFDs. Experiments live in
`bough-static/experiments/`, outside the Bough repositories._

## The answer first

Q1 has three live options now, not two. The addendum's premise holds:
type erasure needs indirection, not a heap. A `dyn Node` can point into
bytes the caller provides, it compiles for all three thumb targets with no
`alloc`, and Miri is clean on it. That makes option (d), the bounded dynamic
engine, real. It keeps the whole API surface, the random-program oracle test
and RFD-0003, and adds one new failure: `construct` can run out of slots.

The static engine, option (b), is smaller and faster to compile by a wide
margin: on a 300-node generated program it is 9 KB of code and 1.4 KB of
RAM against 76 KB and 7.6 KB for the bounded dynamic engine. It pays for
that with a second front end, a second testing harness, and either a subset
of the semantics or a pooled `construct` that brings a collector back. A
builder a human writes without a macro does not survive contact with a
graph of a hundred nodes.

My recommendation is (a), sequenced toward (d): ship the two-tier engine,
put the node arena and every per-transaction buffer behind an internal
storage seam from the first increment, and make the four `Arc`s in the I/O
surface go away now, because (a) needs that anyway to reach a Cortex-M0.
Build (d) when the engine works. Do not build (b). The button-to-LED
milestone runs on (a) with an allocator on both boards you have.

## What I compiled

Toolchain: stable `rustc 1.98.1 (48a229cea 2026-09-01)`, `cargo 1.98.1`;
Miri and `-Zprint-type-sizes` from `rustc 1.100.0-nightly (6bb1652a0
2026-09-22)`. Targets `thumbv7m-none-eabi` (the Due), `thumbv7em-none-eabihf`
(your STM32F303 Discovery) and `thumbv6m-none-eabi` (the G071RB you would
buy). Release profile is `opt-level = "s"`, LTO, one codegen unit,
`panic = "abort"`. A library build only compiles what is already concrete,
so every engine is also linked into a bare-metal image in `firmware/`, which
forces the generic code through codegen for the target.

| Experiment | What it is | v7m | v7em | v6m |
|---|---|---|---|---|
| E0   | The handoff's `no_std` + `alloc` stub, rebuilt                                                   | builds         | builds         | fails: `alloc::sync` twice (E0433), `AtomicU32::fetch_add` (E0599)  |
| E1   | Bump region: `&'g mut dyn Node` into caller bytes                                                | builds, links  | builds, links  | builds, links                                                       |
| E2   | Fixed-size erased slot, free list, generation                                                    | builds, links  | builds, links  | builds, links                                                       |
| E3   | The static engine by hand: flagship, a fan-in/fan-out/loop program, an interrupt-fed input slot  | builds, links  | builds, links  | builds, links                                                       |
| E4   | A static builder over a heterogeneous list, no macro                                             | host only      | host only      | host only                                                           |
| Q10  | One random program at 30, 100 and 300 nodes, emitted three ways                                  | builds, links  | builds, links  | builds, links                                                       |

E0 also shows the replacement for the handle flags: `AtomicBool::store` compiles on
v6m. The M0 has 32-bit load and store; what it lacks is read-modify-write.

### E1 and E2: erasure without a heap

`dyn-core` is a dynamic engine small enough to read: nodes behind `dyn Node`,
cross-node reads through `&mut dyn Table` with `core::any::Any` downcasts, a
split borrow so a node can read its neighbours while it is being evaluated.
It has no unsafe code. It knows nothing about where nodes live; a `Storage`
trait with three methods, `place`, `entries` and `entries_ref`, is the whole
seam, and E1 and E2 are two implementations of it. The same flagship,
`input → accumulate → map_cell → listen_cell`, runs on both.

E1, `e1-bump/src/lib.rs`: `Region::place<T>` splits aligned bytes off a
`&'g mut [MaybeUninit<u8>]`, writes the value and hands back `&'g mut T`,
which coerces to `&'g mut dyn Node`. Two unsafe blocks: the write, and
`drop_in_place` in the storage's `Drop`. It reclaims nothing.

E2, `e2-slot/src/lib.rs`: a `Slot<N>` is `N` bytes at alignment 8 inside an
`UnsafeCell`, beside `Option<fn(*mut u8) -> *mut dyn Node>` and a `u32`
generation. `put::<T>` checks size and alignment in an inline `const` block,
writes the value, and stores `as_node::<T>`, a monomorphized function that
casts the address back to `*mut dyn Node`. The compiler supplies the vtable,
so `drop_in_place` works through the same pointer. The function pointer
does not need to be `unsafe fn`: casting a pointer is safe, only
dereferencing it is. Four unsafe lines, one per operation: write, shared
deref, exclusive deref, drop.

The invariants, as written in the crate:

1. `as_node` is `Some(as_node::<T>)` exactly when the bytes hold an
   initialized, undropped `T`, and `T` is the type `put` was called with.
2. `size_of::<T>() <= N` and `align_of::<T>() <= 8`, checked at
   monomorphization.
3. The bytes live in an `UnsafeCell`, because a read-through node writes its
   memo through `&dyn Node` obtained from `&Slot`.
4. `clear` takes `as_node` before `drop_in_place`, so a panicking destructor
   leaks rather than double-drops.
5. A slot is neither `Send` nor `Sync`, whatever it holds, because the
   erased `T` may be neither. A `Threaded` graph needs a slot type whose
   `put` requires `T: Send`. Without the `PhantomData<*mut ()>` marker the
   compiler would call a slot `Send`, since bytes and a function pointer are.

Tests on the host: E1 three, E2 four (flagship, exhaustion returns `Err`,
free-and-reuse with a drop counter, zero-sized and maximally aligned
nodes). All pass under Miri with Stacked Borrows and with
`-Zmiri-tree-borrows`. Invariant 3 is not theory: the negative control,
the same slot with a plain field in place of the `UnsafeCell`, is undefined
behaviour the first time a listener reads the memoized cell:

```
error: Undefined Behavior: trying to retag from <133006> for SharedReadWrite permission
at alloc42416[0x62], but that tag only grants SharedReadOnly permission for this location
  --> e2-slot/src/lib.rs:80:23
80 |         Some(unsafe { &*ptr })
...
3: <dyn_core::ListenCell<bool, ...> as dyn_core::Node>::dispatch
```

The oversized-node error, from a user crate whose `map_cell` captures a
64-byte table with 32-byte slots (`results/e2-too-big-user.txt`):

```
error[E0080]: evaluation panicked: node state is larger than the slot
   ... evaluation of `e2_slot::Slot::<32>::put::<dyn_core::MapCell<u32, u8,
   {closure@e2-slot/examples/too_big.rs:12:45: 12:53}>>::{constant#0}` failed here
note: the above error was encountered while instantiating
   `fn Slot::<32>::put::<MapCell<u32, u8, {closure@e2-slot/examples/too_big.rs:12:45: 12:53}>>`
   --> e2-slot/src/lib.rs:151:12
```

Legibility: the message says what went wrong, and the only pointer to the
user's code is the closure's file and line embedded in a type name. It gives
neither size, because a `const` panic cannot format a number on stable. Worse,
`cargo check` passes; the error appears only on `cargo build`, so
rust-analyzer never shows it. Any size check made at monomorphization has
this property; I did not test whether `inline_dyn`'s does.

Sizes on thumbv7m, from `-Zprint-type-sizes`: the flagship's four nodes are
1, 16, 4 and 8 bytes. E1 needs 32 bytes of region (with padding) plus four
8-byte table entries. E2 needs four 24-byte slots. Code, from `llvm-size` on
the linked images: E1 1788 bytes, E2 1970, E3 122.

### E3: the static engine, written the way a generator would write it

`e3-static/src/lib.rs`. The library part is `Hold<A>` (value and slot),
`Memo<B>` (an inline `OnceCell`) and `InputSlot<A>`. The generated part is a
struct per program, with nodes as fields and the transaction as a function.
There is no node table, no `dyn`, no marking walk. The affected set of each
input is resolved by the generator, and "not marked" is `None`. The second
program, `blinky`, has a `share` feeding two accumulators, a `merge`, a
`lift` over two cells, and a loop, a hold whose input snapshots the hold.
Its test checks that the loop reads the hold's pre-instant value and that a
press and a tick in one instant merge with the left side winning.

The input slot is the one the handoff describes: one `Option<A>` per input
behind a critical section, folded in place with the coalescing function,
taken by the main loop, one transaction per pending input. The critical
section is two lines of `asm!` (`mrs PRIMASK`, `cpsid i`, `cpsie i`), with
no crate and no atomics, and it links for the M0. A `static` needs a
nameable type, so the coalescing function in a static slot is a `fn`
pointer, not a closure. The graph itself, whose type contains closure
types, cannot be named in a `static` on stable either; it lives on the main
loop's stack, which never returns, and only the input slots are statics.

### E4: a static builder a human writes

`e4-builder/src/lib.rs`. Every materializer takes the builder by value and
returns a bigger type, `(Build<Cons<Node, List>>, token)`. A token is a
zero-sized type naming its node's type. Reading a node finds it in the list
by type, with the position inferred, which is `frunk`'s `Selector` trick. It
works: `(count, time).lift(...)`, `send`, `transaction` and `sample` all
compile, and the example prints the right value. Four findings, each
compiled:

1. A closure's signature has to be stated before the graph's type exists,
   so a cell's value type cannot depend on the graph. The first draft put
   it on `CellNode<Graph, Indices>` and `lift`'s closure lost its
   higher-ranked signature ("implementation of `Fn` is not general
   enough"). A separate `Value` trait fixes it. That is the kind of trap
   every new primitive meets.
2. Identity by type fails for nodes of the same type. Two `input::<u32>()`
   build fine and then `send` and `transaction` fail with E0283, "type
   annotations needed", with the suggestion
   `graph.send::<InputNode<u32>, There<There<There<Index>>>, u32>(...)`
   (`results/e4-same-type-inputs.txt`). Closures make most nodes unique, but
   inputs, constants and `never` are not, so the user writes a marker type
   per input.
3. The foreign-graph check disappears. A token from another graph is caught
   only when its type differs, and then as "`Nil: Get<InputNode<u8>, _>` is
   not satisfied" (`results/e4-foreign-token.txt`). A foreign token whose
   type matches resolves silently to this graph's node.
4. A wrong closure is a clean E0631 at the user's line, because the builder
   states the bound. That part of the surface is as good as today's.

And the cost that settles it, from `gen/e4gen.py`, which emits inputs,
accumulators and two-cell lifts: `cargo check` takes 0.05 seconds at 30
nodes and 0.06 at 100, and a deliberate type error at 100 nodes is caught
by it. A debug `cargo build` at 100 nodes takes 38 seconds and 1.4 GB of
memory. At 300 nodes `cargo check` did not finish in 40 minutes, holding
over 2 GB, and I stopped it (`results/e4-scale.txt`). Somewhere between 100
and 300 nodes the type-level graph stops being a program the compiler can
check. The same 300-node shape as generated code, in the next section,
compiles in half a second.

### E5: what a proc macro would emit, with no types

Added after the first round, to test a proc-macro front end that knows the
graph's structure and nothing about its types. `e5-generic/src/lib.rs`
holds the hand-written expansion of `blinky`: every node's type is a type
parameter, the bounds are written in terms of those parameters, and rustc
infers them where the graph is built. The macro's only type knowledge is
structural: both operands of a `merge` share one parameter, and a hold's
input shares the hold's. At the call site it puts every initial value
first, then gives each closure its own `let` in topological order,
through a `sig` helper that ties the closure to type witnesses of its
inputs and returns a witness of its output. So each closure's argument
types are known before its body is checked.

It works. `examples/blinky.rs` passes every closure unannotated,
`|n, m| ((*m as u32 + n) % 3) as u8`, the loop's `|_, on| !on`, listeners
capturing local `Vec`s, and gives E3's results. One structural finding:
stream types appear only in `Fn` bounds, which cannot constrain an impl
(E0207), so the generated struct carries them in phantom fields.

Errors (`results/e5-*.txt`), with the caveat that `macro_rules!` spans glue
at the whole invocation where a proc macro could point at the operator:

1. A listener expecting the wrong type: E0631 at the user's closure, both
   signatures named. Good.
2. A wrong event type, `|n: u16, m|` on a `u32` input: "type annotations
   needed" on `m`. At the right line, wrong about the cause.
3. A `merge` of a `u8` and a `bool`: "expected `W<bool>`, found `W<u8>`" in
   `merge`. The witness type leaks.
4. A loop's initial value of the wrong type: blamed on the `merge`, not on
   the initial value, because inference runs forward in the order the
   macro wrote.

Three of the four cascade into seven or eight more errors about
`transaction`'s bounds, 110 to 130 lines in all. Better than E4, where a
mistake read as "type annotations needed" deep in the list encoding, and
worse than today's library API.

Code size is byte-identical to the concrete form at every size, on v7m and
v6m: inference costs nothing at runtime. Compile time does grow faster than
the concrete form. Whole-`rustc` time on stable for the release build
(`results/e5-scale-rustc.txt`):

| Nodes | Concrete | All-generic | All-generic, bounds scoped per reader |
|---|---|---|---|
| 30 | 0.14 s | 0.17 s | 0.17 s |
| 100 | 0.23 s | 0.53 s | 0.52 s |
| 300 | 0.49 s | 4.3 s | 2.4 s |
| 600 | 1.1 s | 21 s | 12 s |

The time is in type checking and borrow checking, not codegen. Every
function carries the graph's bounds, and the transaction needs all of
them, so the cost grows as about the square of the node count; scoping
each reader's bounds to the nodes it reaches halves the constant but not
the curve. For tens to a few hundred nodes it is seconds and bounded,
unlike E4. Two cautions: the nightly compiler took 9.8 s at 300 nodes
against stable's 3.9 s, and I did not find out why; and a macro that
accepts optional type annotations could emit concrete types where the user
gave them, which moves any graph toward the concrete column.

### Question 10: the same program three ways

`gen/generate.py` emits one random program, linear-stream-correct, over
`input`, `map`, `merge`, `accumulate`, `map_cell`, `lift2` and
`listen_cell`, as an E1 image, an E2 image, and the struct and transaction a
static generator would emit. `gen/measure.py` and `gen/timeit.py` measure
it. Compile time is a real rebuild of the program crate alone, release,
LTO, dependencies built, best of two or three. Cargo 1.98 decides freshness
by content, so `timeit.py` appends a comment before each run; `touch` is not
enough. RAM for E1 is the exact node bytes plus the table; for E2 it is
slots times slot size; for the static engine it is the generated struct.
The slot size came out at 16 bytes at every size, since the largest node in
these programs is an accumulator.

thumbv7m (thumbv7em is within 1% of it):

| Nodes  | E1 compile · code · RAM    | E2 compile · code · RAM    | Static compile · code · RAM  |
|---|---|---|---|
| 30     | 0.39 s · 9.6 KB · 620 B    | 0.41 s · 9.1 KB · 744 B    | 0.17 s · 0.9 KB · 148 B      |
| 100    | 0.85 s · 28.2 KB · 2.1 KB  | 1.03 s · 27.8 KB · 2.5 KB  | 0.25 s · 3.4 KB · 480 B      |
| 300    | 2.29 s · 77.8 KB · 6.2 KB  | 3.56 s · 75.8 KB · 7.6 KB  | 0.50 s · 9.0 KB · 1.4 KB     |

thumbv6m at 300 nodes: E1 73.4 KB, E2 70.1 KB, static 10.7 KB of code, same
RAM, same compile times (`results/q10-*.txt`).

Against the boards, at 300 nodes. RAM: the SAM3X8E has 96 KB of general
SRAM, the F303VC 40 KB plus 8 KB of core-coupled memory, the G071RB 36 KB.
E2's 7.6 KB is 8%, 19% and 21% of those. Flash, which the handoff did not
ask about and which turns out to be the tighter wall: 512 KB, 256 KB and
128 KB. E2's 76 KB is 15%, 30% and 59%.

Read these as ratios, not as a forecast. `dyn-core` has no dependents lists,
no generations on E1, no marking walk, no token checks and no collector, so
a real dynamic engine is bigger in both RAM and code, and the gap to the
static engine is at least this wide. E2 against E1 is a fair comparison:
uniform slots cost 21% more RAM than exact placement at 300 nodes, 1.3 KB.
The code cost of the dynamic engines, about 250 bytes per node, is the
monomorphized node impls behind the vtables. It is not a cost of (d); (a)
pays it too.

## The allocation inventory

Every site in the skeleton at `dfe736b`, then every structure the RFDs imply
that the skeleton does not have yet. "Bounded" is the (d) replacement; "M0"
says whether it needs anything the Cortex-M0 lacks.

| Site | Today | Bounded replacement | M0 |
|---|---|---|---|
| `Graph::set_waker` (`graph.rs:244`) | `Arc<dyn Fn() + Send + Sync>` | `&'static (dyn Fn() + Sync)`; an API change | fine |
| `Listener.alive` (`graph.rs:299`) | `Arc<()>` shared flag | `&'s AtomicBool` into a fixed flag table in the caller's storage; drop is a plain store | fine: store only |
| `Anchor.alive` (`graph.rs:320`) | `Arc<()>` | as `Listener` | fine |
| `Remote.inbox` (`graph.rs:343`) | `Arc` around a `std::sync::Mutex` | `&'s Inbox`, a fixed ring behind a critical section; single producer can be lock-free with load and store | fine |
| Remote units (`graph.rs:382`, RFD-0006) | `Vec<Box<dyn FnOnce() + Send>>` | a fixed ring of E2-style slots holding `dyn FnOnce`; the slot trick erases a unit as well as a node | fine |
| Graph id | `AtomicU32::fetch_add` | the storage's address, unique among live graphs; or an id the caller passes | fine |
| `Tracer.visited` (`trace.rs:17`) | `Vec<Token>` | a mark stack sized to node capacity; each node is pushed at most once | fine |
| `Trace` impls for `Box`, `Rc`, `Arc`, `String`, `Vec`, `VecDeque`, `BTreeMap`, `BTreeSet` | unconditional | behind `alloc` | `Arc` impl needs `alloc::sync`: gate it |
| `Trace` impls for `HashMap`, `HashSet`, `Instant` | unconditional | behind `std` | n/a |
| `std::error::Error` (`error.rs:6`) | std | `core::error::Error`, stable since 1.81; MSRV is 1.85 | fine |
| The `Remote` guard's thread id (RFD-0006) | `std::thread` | see below: on bare metal the question is interrupt context, not thread | fine |
| Node arena (RFD-0003) | `Vec` of nodes | E2 slots with a free list; generation per slot | fine |
| Dependents lists (RFD-0005) | a `Vec` per node, implied | one fixed edge pool, lists threaded through it by index | fine |
| `depends` declarations (RFD-0003) | implied `Vec` | the same edge pool, a second kind of edge | fine |
| Mark stack, on-stack flags, reverse post-order (RFD-0005) | "two reused vectors" | arrays sized to node capacity; bounded by construction, never exhausted | fine |
| Listener closures and dispatch list | implied boxed | E2 slots, or `&'s mut dyn FnMut` in the E1 style | fine |
| `construct` closure (RFD-0002) | implied boxed | an E2 slot | fine |
| A scope's open loops (RFD-0002) | implied `Vec` | a small fixed array per scope | fine |
| Child transaction queue (`split`, `defer`) | implied queue | see question 8: store the iterator, not the items | fine |
| Collection mark bits | implied | one bit per slot | fine |

Two sites have no bounded answer that keeps today's behaviour, and they are
the ones to name. First, `construct` exhausting the slots, below. Second,
the `Remote` guard: it rejects a send from the driver thread during a
transaction. On bare metal an interrupt that fires during a transaction runs
on the same core and is a legitimate sender, so a thread check, or a flag
check alone, would reject exactly the sends the inbox exists for. The guard
has to ask whether the caller is an exception handler, which on Cortex-M is
the IPSR register. That is a small port per architecture, not a blocker.

Three of these sites, the waker and the two handle flags, are also the
reason E0 fails on the M0 under (a). `alloc` exists on thumbv6m;
`alloc::sync` does not, because `Arc` needs compare-and-swap. So (a) reaches
the M0 without `portable-atomic` only if the I/O surface has no `Arc` in it,
which is the same change (d) needs.

## Slot exhaustion

Under (d) `construct` can find no free slot. The semantics' `Execute`
always succeeds, and the transaction is half evaluated when it happens, so
there is no answer that finishes the instant correctly. The policy: a
panic inside the transaction, which poisons the graph under RFD-0005's
existing rule, with `try_send`, `try_transaction` and `try_pump` returning
a new `Exhausted` variant rather than `Poisoned`, so firmware can tell a
capacity bug from a logic bug and reset. Build-time exhaustion, outside
`construct`, is an ordinary build panic. `live_nodes` and a high-water mark
let tests size the storage.

I do not think this is a deviation needing its own RFD, and this is where
it differs from a static subset. A subset cannot express some programs, so
it gives different answers from the semantics' point of view. Exhaustion
never gives a different answer; it gives no answer, loudly, the same way
the `std` engine aborts on allocation failure and nobody calls that a
semantics change. What it does need is text in RFD-0003, because unlike
out-of-memory on a desktop it is reachable by design, and an error variant
in RFD-0005's table. The oracle test cannot see capacity. A property test
can: run the random programs with storage sized to the high-water mark and
assert no exhaustion, then sized one below and assert `Exhausted`, never a
wrong value.

The slot size `N` is a second, milder limit: a node larger than `N` does
not compile. That restricts programs at build time, so the user raises
`N` or moves the big value into a cell of its own. With one large node per
graph, uniform slots waste the most; size classes, two or three slot arenas,
are the answer if it bites, at the cost of a second free list.

## The handoff's questions

**1. A static graph as Rust types.** Three identities, and only one scales
(E5 tests the proc-macro form of it).
A zero-sized token naming the node's type, looked up by type (E4), is
writable by hand, but identical types collide, the foreign-graph check
vanishes, and the type checker falls over somewhere between 100 and 300
nodes. A const-generic index has the same lookup problem plus arithmetic on
const generics, which is unstable. A field name, emitted by a macro or a
build script (E3, and the Q10 generator), gives ordinary errors pointing at
generated code, compiles 300 nodes in half a second, and is what Copilot,
Lustre and Heptagon all do. So the static engine is a compiler, not a
library of types.

**2. The builder.** With a field-path identity nothing of `Build` survives.
The adapters and chains survive as a notion, since fusion is exactly what
the generator does to a linear region, but not as the `Source` trait's
types: the generator sees the chain as syntax. The materializers, `Build`,
`CellLoop` and `StreamLoop`, tokens as data and `Graph::build(|b| ...)` all
go. A proc macro over a `graph! { ... }` block sees syntax, not types, so
`let` bindings of tokens, helper functions that build graph over `Build<M>`,
and loops closed in another function become macro-language features to
reimplement. That is the second product.

**3. `construct` and the switches.** Three findings. First, `switch_stream`
and `switch_cell` over tokens that exist at build time need no allocation:
the candidate set is every node of the right type, the generator knows it,
and relinking is a `match`. They survive. Second, dropping `construct`
makes the tier a strict subset and needs a deviation RFD; the oracle can
still check every program without `construct`. Third, a pool per construct
shape: a construct closure builds one sub-graph type each time (an enum if
it branches), so its instances are a fixed array of that type, homogeneous,
no erasure. But instances die, tokens to them travel inside cell values, and
freeing one needs `Trace`, generations and a collector over the pool. That
is RFD-0003 again, per pool. Pre-built alternatives with a reset on switch,
Lucid Synchrone's answer, match `Execute` only when a construct fires
exactly at its switch and each instance is used once; Sodium's hold inside
`construct` starts at its creation instant, and a pre-built one has been
running since build. So it is a subset too.

**4. The memory model.** Without `construct` it collapses completely, and E3
shows it: no arena, no generations, no `Trace`, no collection, no stale
tokens, no anchors. Everything built is alive forever, which is the
semantics' answer too, since nothing can become unreachable. With pooled
`construct`, all of it comes back inside each pool. Under (d) nothing
collapses and nothing needs to: `construct` draws a slot, collection frees
one, and RFD-0003 stands as written apart from exhaustion.

**5. The transaction over a static type.** E3's `send_presses` and
`blinky::transaction` are the sketch, and they compile for all three
targets. Evaluation is straight-line code in topological order; an
unaffected node is a `None` that falls through, so the "bitset per input"
is not even a runtime value. One function per input (smaller per call,
code grows with inputs times region) or one general function over a
`Sends` struct (one copy of the code, `Option` checks everywhere); the Q10
numbers use the general form. Memos are inline `OnceCell`s. Under (d) this
question becomes RFD-0005's transaction over arrays sized to capacity, as
the inventory says.

**6. Listeners.** Declared at build in the static engine, since a listener
closure is part of the type; E3 does this. But runtime `listen` needs no
heap: a node can keep a fixed table of `&'s mut dyn FnMut(&A)`, E1's
technique, so I/O code registers a closure it owns in a `static` or on the
main stack. A `Listener` is then an index and an `&'s AtomicBool`; drop
stores `false`, `unlisten` is drop, `keep` forgets the handle. The same
answer serves (d), with E2 slots as the alternative when the closure should
live in the graph.

**7. The I/O edge.** One `InputSlot<A>` per input, as in E3: written under a
critical section, folded with the input's coalescing function, taken by the
main loop, one transaction per pending input, never two inputs in one
instant. The critical section is a PRIMASK save and `cpsid`, which the M0
has. A lock-free single-producer slot needs only load and store, so it
works on the M0 as well; `heapless::spsc` confirms that by hardcoding
load-and-store support for thumbv6m. `remote.transaction` has no home in a
per-input slot. In the static engine I would declare a multi-input unit at
build, a typed slot over a tuple of `Option`s with each component folded by
its own input's function. Under (d) the dynamic unit survives unchanged,
as a fixed ring of erased `FnOnce` slots, with `Full` on overflow.

**8. Child transactions.** A bounded queue of items is the wrong shape.
`split`'s event is an `IntoIterator`; store the iterator in the split node
and pull one item per child transaction. Two splits in one instant share
child indices, which is exactly advancing both iterators in lockstep. No
item is ever copied into a queue. What is bounded is nesting: a child
transaction can fire another split, so the engine keeps a stack of active
splits. If no split can reach itself through child transactions, the depth
is at most the number of split nodes, which the static generator can check
and (d) can check at close. If one can, the stack gets a fixed depth and
overflow poisons, like exhaustion. `defer` is a split of one, an `Option`.
I have not compiled this one; it is the next experiment if `split` matters
on embedded.

**9. Testing the static engine against the oracle.** Three options, costed
with the Q10 numbers.
- Generate Rust source for random programs and compile them in the
  harness. At 0.2 to 0.5 seconds of compile per program, or less if a
  hundred programs share one crate, a few hundred cases fit in a
  CI job. Shrinking works but recompiles on every step. This is Copilot's
  method: compile the C, run it, compare against a Haskell reference. The
  cost is a second emitter for the generator the oracle test already needs,
  one to two weeks.
- A fixed corpus of hand-written programs. Cheap, and a regression suite
  rather than a test of the semantics.
- An interpreter of the static description. That tests the interpreter,
  not the generated code, and the interpreter is the dynamic engine.
The first option is the one that honours "the oracle is the test". Under
(d) the question dissolves: the storage changes, the test does not.

**10. Size regime and costs.** The static engine serves tens to a few
hundred nodes and handles 300 easily as generated code: half a second,
9 KB, 1.4 KB. As a hand-written type-level builder it stops at about a
hundred. The errors are the builder's weak point: E4's ambiguity and
foreign-token messages are unreadable without knowing the list encoding,
and `#[diagnostic::on_unimplemented]` helps with the second but not the
first. Bevy's issue 12377 is the precedent: its tuple errors were bad enough
to need that attribute.

**11. A backend interface.** Building the static engine does not argue for
one, because it is not a backend in that sense: it replaces the program
representation, not the node store. Its interface is a compiler's, the
description of a graph, not a trait the semantics could be written against.
(d) argues for something much narrower: E1 and E2 share `dyn-core`
unchanged through a three-method `Storage` trait. That seam is internal,
costs nothing to keep from the first increment, and is the only thing (d)
needs from the engine. So the recommendation not to design a public backend
now stands, with one amendment: the arena goes behind a `pub(crate)`
storage trait from day one.

**12. Prior art.** Checked with sources by a research pass; the details
are in the conversation, the shape here.
- Copilot compiles a stream DSL to constant-memory C. Delays are finite
  prefixes whose length is the buffer. There are no streams of streams. Its
  generated C is tested against Haskell reference functions with
  QuickCheck, and `copilot-verifier` proves the equivalence.
- Lustre rejects instantaneous cycles and requires a `pre` or `fby` on every
  loop. That is our hold rule. Its clock calculus is what bounds memory.
  Heptagon's generated code is a `reset` and a `step` over explicit state,
  which is E3's shape. Dynamic reconfiguration in that family is modular
  `reset` and automata, where one mode's equations run per instant: finite,
  pre-built alternatives. That is question 3's second option, with the same
  subset problem.
- Elm before 0.17 ruled out signals of signals on purpose. The PLDI 2013
  paper's reason is that a signal created later that folds over an input
  needs either the input's whole history or two identical definitions with
  different values. That is exactly the problem `hold`'s creation instant
  solves, and a principled citation for why a static tier cannot have
  `construct` cheaply.
- `frunk`'s `Selector` is E4's lookup. Bevy's tuples stop at 16 elements
  and nest beyond that.
- `heapless` gives `Vec` and `spsc` with no atomic read-modify-write, M0
  included. Its pools need LDREX/STREX and exclude the M0. `static_cell`'s
  `make_static!` needs nightly, and it needs `portable-atomic` plus a
  `critical-section` implementation on the M0.
- Inline trait objects: `smallbox` falls back to the heap and always links
  `alloc`, so it is out. `stack_dst` is `no_std` without `alloc`, checks
  size at runtime and returns `Result`. `inline_dyn` checks at compile time
  on nightly, or through a macro on stable. `inplace-box` and `static-box`
  are nightly. E2 is `inline_dyn`'s idea in about 70 lines on stable, and needs no
  dependency.
- I found no `no_std` Rust FRP crate with a static graph. The search was
  shallow.

## What survives, changes or dies

The semantics' primitives, and the operations built on them, in each tier.

| Primitive | (a) | (d) | (b) without `construct` | (b) with pooled `construct` |
|---|---|---|---|---|
| `never`, `constant`, `input` | survives | survives | survives; inputs need markers in a type-level builder | survives |
| `map`, `filter`, `filter_map`, `map_to`, `snapshot`, `gate`, `once` | survives | survives | survives, fused by the generator | survives |
| `merge`, `or_else` | survives | survives | survives | survives |
| `hold`, `accumulate`, `accumulate_mut`, `scan` | survives | survives | survives | survives |
| `map_cell`, `lift` | survives | survives | survives | survives |
| `steps`, `steps_with_current` | survives | survives | survives | survives |
| `switch_stream`, `switch_cell` | survives | survives | survives over build-time tokens | survives |
| `construct` (`Execute`) | survives | survives; can exhaust | dies: a deviation | changes: per-shape pools, a collector per pool |
| `split`, `defer` | survives | survives; bounded nesting | survives; bounded nesting, checkable | same |
| Loops | survives | survives | survives | survives |
| `listen`, `listen_cell`, `listen_steps` | survives | survives; fixed tables | declared at build, or `&mut dyn` tables | same |
| `Anchor`, collection, `Trace`, generations | survives | survives | dies: nothing is ever garbage | returns inside pools |
| `Remote`, `pump` | survives | survives; fixed ring | changes: typed per-input slots, units declared at build | same |
| The random-program oracle test | survives | survives | changes: source generation | same |

## What each option costs

**(a)**, two tiers. Cost now: the M0 needs the `Arc`s out of the I/O
surface, and the arena behind an internal seam. Those are days, and the
second is free if it is there from the first increment. RFD-0006 changes
for the waker; RFD-0002 does not change.

**(d)**, a third tier on top of (a). About 250 lines of storage code for
slots, the edge pool and the flag table, bounded versions of the inbox and
the child stack, the exhaustion policy and its property test, and
`Threaded` slots. I estimate two to three weeks after the engine works.
API surface: a capacity type parameter or storage argument on
`Graph::build`, `&'static` in place of `Arc` at `set_waker`, `Exhausted` in
three error enums, `Full` on remote sends. RFD-0003 gets a section, not a
rewrite.

**(b)**, a static tier. E5 shows the front end is feasible without type
information: a proc macro that emits the all-generic form compiles 300
nodes in 2 to 4 seconds and gives errors at the user's closures most of the
time. What it does not remove is the rest of the cost. A proc-macro front end with its own syntax for
tokens as data, helpers and loops across functions; a code generator;
switch over candidate sets; pools if `construct` is wanted, which brings a
collector back; a source-generating oracle harness. I estimate eight to
twelve weeks, plus a deviation RFD if `construct` is dropped. API surface:
a second one. What it buys is real: five times less RAM and eight times
less code on the programs measured here, and a transaction with no
indirection at all. It is the tier to revisit if flash on a 128 KB part
turns out to be the wall, and the Q10 generator plus E3 are a working start
on it.

## What I did not do

No `split` experiment (question 8 is reasoned, not compiled). No hardware
run; everything links, nothing has executed on a board. `dyn-core` is a
stand-in, not a design for the engine, and its numbers are floors. The
prior-art pass was a web search with citations, shallow on Esterel and on
existing crates.

## Keeping the experiments

`bough-static/experiments/` is a scratch workspace outside the Bough
repositories: `e0-alloc-baseline`, `dyn-core`, `e1-bump`, `e2-slot`,
`e3-static`, `e4-builder`, `firmware`, `gen/` with the generators and
timing scripts, and `results/` with the recorded errors and measurements.
The one piece worth keeping is `e2-slot`, as the seed of (d)'s storage and
as the reference for its unsafe invariants. Where it goes is your call.
