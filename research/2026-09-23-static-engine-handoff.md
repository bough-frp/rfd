# Handoff: exploring a static, no-alloc core for Bough

_Written 2026-09-23 from the Bough design session
(https://claude.ai/code/session_01XZ4jSiBN5gqeLQL1Q79TYd). Transient working
state, not a record: nothing here has authority over the RFDs._

## Your goal

Explore what designing and implementing Bough's core as a **static FRP
engine that runs without an allocator** would look like, and report back.
The output is a written exploration with compile experiments, not code in
the Bough repositories and not an RFD. It feeds one open question in a
design round Zefira is answering now (see "The question this informs").

You are exploring, not deciding. Zefira decides, and she decides in
conversation: put questions to her in numbered rounds with a recommendation
per question, five at a time, and wait for answers. She answers in the
compact form `Q1: a, Q2: agree`.

## What Bough is, and where to read the design

A functional reactive programming library in Rust that implements the
Sodium denotational semantics, version 1.1, exactly, with an API that makes
it structurally impossible to mix building FRP logic with driving it from
I/O. Everything below is a pointer; the design itself lives here:

- The RFDs, on the branch under review:
  <https://github.com/bough-frp/rfd/pull/2>, head
  `RadicalZephyr:rfd/0002-api-shape` at `780b877`. Read `GLOSSARY.md` first,
  then RFD-0002 (API), RFD-0003 (memory model), RFD-0004 (value model),
  RFD-0005 (transaction protocol), RFD-0006 (I/O edge). RFD-0001 is merged
  on `main` and carries the standing policies: exact fidelity, the
  performance bar, the oracle, naming.
- The API skeleton, every signature with `todo!()` bodies, with the RFD
  examples as `no_run` doc tests and the guarantees as `compile_fail` doc
  tests: <https://github.com/bough-frp/bough/pull/10> at `44a2184`, plus
  the follow-up stacked on it at
  <https://github.com/RadicalZephyr/bough/pull/1> at `dfe736b`, which adds
  the tuple `lift`, the cell stream views and `Leaf<T>`. Read the skeleton
  as the current API shape; it is what your static design has to be
  compared against.
- The semantics, the thing both engines are held to:
  `denotational/Reactive/Sodium/Denotational.hs` in
  <https://github.com/SodiumFRP/sodium>, 122 lines. A markdown rendering of
  the accompanying document, with section numbers, is vendored under a BSD
  licence at
  <https://github.com/RadicalZephyr/bevy-sodium/tree/main/docs/reference/sodium>.
- The requirements and the design history, which Zefira has as files
  (`REQUIREMENTS.md`, `PLAN.md`, `GRILLING-TRANSCRIPT.md`); ask her for
  them if you need the reasoning behind a settled decision. Do not reopen a
  settled decision without a new fact.

## The shape you are comparing against

The dynamic engine, in one paragraph, so you can see what a static one
would replace. Nodes live in one arena owned by `Graph`, addressed by
tokens of index, generation and graph id; `Stream<A>` is move-only and
linear, `Shared<A>`, `Cell<A>` and `Input<A>` are `Copy`. Adapters (`map`,
`filter`, `filter_map`, `map_to`, `snapshot`, `gate`, `once`) take no
context and return chain types such as `Filter<Map<S, F>, P>`; a chain fuses
into one node with one monomorphized closure when a materializer takes it
with `&mut Build<M>`: `hold`, `accumulate`, `accumulate_mut`, `scan`,
`share`, `node`, `merge`, `or_else`, `split`, `defer`, `construct`,
`switch_stream`, and on cells `map_cell`, `switch_cell`, `steps`,
`steps_with_current`, with `lift` on a tuple of cells. Loops are flat:
declare a forward token, hand it anywhere, close it later in the same scope.
`construct` is the semantics' `Execute` and the only way logic is added
after build. A transaction is begin, mark (a depth-first walk over
dependents lists), evaluate (a flat loop in reverse post-order), commit
(holds move their slot into their value, read-through memos are cleared,
switches relink), dispatch (listeners, post-commit, no graph access), then
child transactions for `split` and `defer`. Read-through cells memoize in a
`core::cell::OnceCell` cleared at commit. Liveness is tracing collection
from explicit roots; `Trace` finds tokens inside cell values. I/O reaches
the graph through `Graph` (`send`, `transaction`, `listen*`, `sample`,
`anchor`, `pump`, `remote`, `set_waker`) and, from other threads, through a
`Remote` that queues units into an inbox the driver pumps.

## Why a static engine is a different engine, not the same one with growth removed

This is the fact that put the question on the table. Every node holds a
fused closure of its own type. A graph of heterogeneous nodes in one arena
needs type erasure, which is a `Box<dyn Fn>`, which is an allocation at
node creation. Without an allocator, the graph's shape has to be known to
the compiler: the nodes become fields of a generated type, the edges become
type-level references, and the transaction becomes a generated function
over that type. That is the "compiled design" the session deferred in the
first design round (Q21) because it costs weeks and makes every dynamic
region a special case, and it is exactly what a no-alloc tier would need.
Zefira wants to see what it looks like before deciding whether it exists.

## The question this informs

From the current grilling round, verbatim:

> **Q1 - The no-std tiers**: The note wants "bounded is the core, growth is
> the opt-in", with `alloc` and `std` as features that lift restrictions.
> \[...\] Options: (a) two tiers, `no_std` with `alloc` as the unconditional
> core and `std` as a default-on feature, plus a policy that the engine
> allocates only at build and inside `construct`, never per transaction;
> (b) three tiers with a no-alloc core, which means designing the static
> engine now; (c) `std` only, embedded later.
>
> Recommended: (a). It honours the insight that matters, no allocation while
> the graph is not growing, with the arena reserved at init and every
> per-transaction structure reused. Both your boards have allocators.

Your exploration should let Zefira choose between (a) and (b) with the
costs of (b) known: what the static engine can express, what it cannot,
what it costs to build and to use, and whether it can share a surface with
the dynamic engine or is a second product.

## Key questions to answer

Answer these with compile experiments where a claim is about Rust. The
session's habit is "compile the claim": three Rust claims in an early RFD
draft were wrong and a stub found all three.

1. **What is a static graph as Rust types?** A chain type is already a
   static description of a linear region. Extend it to a whole graph with
   fan-in (`merge`, `lift`, `snapshot` read two or more nodes), fan-out
   (`share`, one node read by several), and a loop (a hold whose input
   reads the hold). Type-level references need an identity: a zero-sized
   marker type per node, a const-generic index, or a field path. Which one
   gives readable errors and a builder a human can write?

2. **What builder produces it?** Today materializers take `&mut Build<M>`
   and return tokens; the graph grows behind a mutable reference and the
   closure's return value is the edge. A static builder must accumulate
   types, so every materializer changes the type of the builder, which
   forces by-value chaining (`let (b, count) = ...hold(b, 0)`) or a
   typestate builder over a heterogeneous list. Determine whether any part
   of the current surface survives: the adapters and chains probably do,
   the materializers and `Build` probably do not. Say precisely which.

3. **What happens to `construct`, `switch_stream` and `switch_cell`?**
   Runtime construction allocates by nature. Options to evaluate: drop
   them in the static tier, which makes the tier a strict subset of the
   semantics and collides with RFD-0001's exact-fidelity policy; switch
   over a finite, statically enumerated set of pre-built alternatives, so
   peak live nodes are bounded because all alternatives exist at once; or
   pools of pre-sized slots per node shape drawn by `construct`. For each,
   what the oracle test can still check.

4. **What happens to the memory model?** Without `construct` there is no
   garbage: no arena, no generations, no `Trace`, no collection, no stale
   tokens. Verify that this really collapses the whole of RFD-0003 in the
   static tier, and what survives if bounded `construct` (question 3) is
   allowed.

5. **How does a transaction run over a static type?** The mark walk over
   dependents lists becomes, for a static topology, a precomputed bitset of
   affected nodes per input; evaluation becomes a generated walk in a fixed
   topological order that skips unmarked nodes. Cells keep an inline
   `OnceCell<A>` memo, no box needed because the type is known. Sketch the
   generated function for the flagship example and show it compiles for
   `thumbv7m-none-eabi` with `#![no_std]` and no `alloc`.

6. **Listeners without allocation.** Listeners are RAII handles registered
   at runtime. A static tier either declares them at build, or gives each
   node a fixed-capacity listener table. Which, and what `keep` and
   `unlisten` become.

7. **The I/O edge without allocation.** The inbox becomes one typed slot
   per input, written from an interrupt handler under a critical section,
   folded in place with the input's coalescing function, drained by the
   main loop as one transaction per pending input. There is no queue, so
   `remote.transaction`, the multi-input unit, has no home; decide whether
   a typed tuple slot replaces it or the static tier drops it. The
   simultaneity rule from the current round: a drain never puts two inputs
   into one instant, because unrelated inputs made simultaneous by timing
   is unsound (`merge` combines them). Cortex-M3 and M4 have 32-bit atomics
   and compare-and-swap; Cortex-M0 has none, so anything needing an atomic
   read-modify-write is out for M0 without `portable-atomic`.

8. **Child transactions without allocation.** `split` and `defer` schedule
   children `t ++ [n]`, unbounded in the semantics. A static tier needs a
   bounded queue per split with a stated overflow policy, or drops them.

9. **How is a static engine tested against the oracle?** The dynamic
   engine is property-tested with random programs generated at runtime. A
   type-level program cannot be generated at runtime. Options: generate
   Rust source for random programs and compile them in a test harness;
   a fixed corpus of hand-written programs; or an interpreter of the static
   description that the property test drives. Cost each, since "the oracle
   is the test" is standing policy.

10. **Size regime and costs.** A graph of ten thousand nodes as one type is
    not a thing. State the regime the static tier serves (tens to hundreds
    of nodes on a microcontroller), and measure compile time, code size and
    error legibility on a graph of that size.

11. **Does the static engine argue for a backend interface?** A separate
    Bevy research note asks for the graph representation to be a backend
    behind an interface the semantics are written against. The session's
    current recommendation is no, not now: extract a trait from a working
    engine later rather than guess its shape now. A static engine is a
    second backend. Say whether building it changes that recommendation,
    and what the interface would have to expose.

12. **Prior art.** Look before designing, and verify rather than recall:
    Galois's Copilot, a stream DSL that compiles to constant-memory C for
    embedded runtime verification; the synchronous dataflow languages
    Lustre and Esterel, whose static scheduling and bounded memory are the
    theory this tier would be reinventing; Elm's pre-0.17 static signal
    graph; and in Rust, the typestate and heterogeneous-list builders
    (`frunk`, Bevy's system tuples) and the embedded allocation patterns
    (`heapless`, `static_cell`). Note what each gives up.

## Constraints you do not reopen

- Exact fidelity to the Sodium 1.1 semantics; the oracle property test is
  the acceptance test. A static tier that is a subset must say so as a
  deviation, which needs its own RFD.
- Building FRP logic and driving it with I/O stay structurally separate:
  nothing adds logic after build except `construct`, listeners have no
  graph access, and I/O code sees only tokens or their static equivalent.
- The naming policy in RFD-0001 and the glossary's vocabulary. Use the
  glossary's words; the domain-modeling skill's format rules are in force.
- The engine crate has zero runtime dependencies by default; a feature-gated
  optional dependency is a question for Zefira, not a decision you make.
- No implementation in the Bough repositories. Experiments go in a scratch
  crate; if one is worth keeping, say so and Zefira decides where.

## Facts verified in the session you can rely on

- A `#![no_std]` + `alloc` stub of the memo (`core::cell::OnceCell`), the
  waker (`alloc::sync::Arc<dyn Fn() + Send + Sync>`), a boxed node
  closure and an `AtomicU32::fetch_add` graph-id counter compiles for
  `thumbv7m-none-eabi` and `thumbv7em-none-eabihf`. The same source fails
  on `thumbv6m-none-eabi`: `alloc::sync` does not exist there and
  `AtomicU32` has no `fetch_add`.
- The skeleton's standard-library surface is small: `std::sync::Mutex` and
  `Box<dyn FnOnce + Send>` in the inbox, `HashMap`/`HashSet`/`Instant` in
  `Trace` impls, `std::error::Error`, and the thread id the `Remote` guard
  records. Everything else is `core` or `alloc` under another path.
- Zefira's boards: an STM32 Discovery (Cortex-M4 or M3 depending on the
  model; ask which) and an Arduino Due, SAM3X8E, Cortex-M3. The first
  embedded milestone is a button press through a tiny graph to an LED,
  bare metal.
- The no-std note's insight that the session adopted, pending her answer:
  the memory-model principle "never pay for allocation unless the graph is
  growing" already implies that a graph that is not growing needs no
  allocator at runtime. In the dynamic engine that becomes "allocate only at
  build and inside `construct`, reuse every per-transaction structure",
  which is what option (a) in Q1 promises.

## How Zefira works

- Conversation before artifacts. State back what she is trying to do and
  why, get agreement, then write. An early idea stays a conversation.
- Push back on weak reasoning, hers and yours; several decisions in this
  design changed because a recommendation was argued down.
- Verify, do not assert: propose a concrete experiment and run it.
- Increments reviewable in about five minutes; documents come out nearly
  right in one pass, and if one is being edited for more than a turn or
  two, stop and go back to talking.
- Her pronouns are she/her.

## What to hand back

One dated note, in her voice (short sentences, spelled-out words, no
hedging), with: the design sketch answering questions 1 to 8; the compile
experiments as source, output and toolchain version, the way
`bevy-sodium`'s `docs/decisions/README.md` records experiments; the list of
semantics primitives that survive, change or die in the static tier; the
testing strategy from question 9; the size regime and measured costs from
question 10; and a recommendation on Q1, (a) or (b), with what (b) costs in
weeks and in API surface. The note goes to the Bough design session, which
folds it into the grilling round.

## Suggested skills

Call these with the Skill tool if they are listed in your session; the set
varies.

- `anthropic-skills:grilling` for any round of questions to Zefira; it is
  the format the design has used throughout.
- `anthropic-skills:domain-modeling` if your sketch needs new vocabulary;
  the glossary is `GLOSSARY.md` in the `rfd` repository and its format rules
  apply to any term you add.
- Earlier handoffs suggested `sodium-frp`, `tdd` and `engineering:*` skills;
  none was available in this session. Check before assuming.
