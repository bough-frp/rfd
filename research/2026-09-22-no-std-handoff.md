# Handoff: keeping a no-std mode viable in Bough

_From a claude.ai voice chat, 2026-09-22. For the conversation designing Bough._

## Why this came up

The user wants to test the claim in Blackheath & Jones's *Functional Reactive Programming* that FRP can drive embedded systems, by eventually running Bough as the main runtime on bare-metal hardware (first an STM32 Discovery board, later her own 3D-printer controller shield on an Arduino Due / SAM3X8E; both Cortex-M). Bough is still early in design: decision records are being drafted and the Sodium denotational-semantics oracle port isn't written yet. That makes now the cheap moment to make no-std a first-class design driver instead of a retrofit.

Everything below is notes and candidate constraints, not decisions. Grill them in the Bough conversation before anything lands in `docs/decisions/`.

## Existing Bough design this builds on

See the Bough repo for details; listed here only as anchors.

- Single-threaded core with a global transaction step (motivated by research suggesting threading and coordination overhead outweighs the gains for typical in-graph FRP workloads).
- Graph stored as a backing array (currently a `Vec`) addressed by generational indices. User-facing handles are index + generation + graph id: `Send + Sync + Copy` tokens. The real machinery lives on a separate object.
- The `Remote` struct, designed so GTK-rs callbacks (thread affinity) and Tokio tasks can push input into the graph. It is effectively a mailbox.
- The memory-model RFD: never pay for allocation unless the graph is genuinely growing.
- Plan: port Sodium's denotational semantics to Rust as an oracle for property-testing Bough.

## Insights to carry over

### 1. Bounded is the core; growth is the opt-in

The memory-model RFD principle already implies no-std: a graph that isn't growing isn't allocating, and a graph that isn't allocating doesn't need std. So the polarity should be that a fixed-capacity graph is the default core, and `std` (or `alloc`) is a feature that *lifts* the growth restriction. Don't layer no-std as a restriction onto a core that assumes `Vec` everywhere; retrofitting bounds is where these efforts usually die.

### 2. Single-threaded core + global transaction = embedded main loop

On a microcontroller there's usually one execution context for main logic anyway, so the single-threaded core is native there, not a compromise. The transaction step maps directly onto the classic control loop: drain inputs, run a transaction, propagate, drive outputs, repeat.

### 3. Interrupts are a third caller of `Remote`

GTK callbacks, Tokio tasks and interrupt handlers all need to hand input to a single-threaded core without racing an in-flight transaction. The answer is the same for all three: a remote/mailbox that the core drains at transaction boundaries. The humblest embedded version is a fixed slot per input, written from the ISR and drained by the main loop.

Main risk: `Remote` is conceptually no-std-ready but reaches for std under the hood (see the audit checklist below).

### 4. Bound input bursts with coalescing (the user's insight)

Problem: each external push is its own transaction. If an input fires several times before the core drains it, a per-push queue grows without bound.

Sodium already answers a structurally identical problem. A stream fires at most once per transaction, and merging streams that fire simultaneously requires either picking one or supplying a coalescing function. Treat the input boundary as one more merge point: each bounded input declares a coalescing function (or a policy such as latest-wins), and a burst folds into that input's single pending slot in place.

Result: a fixed graph layout plus a fixed set of inputs with one slot each gives a fully static memory layout. The semantics supply the memory model, which is decent evidence for the Blackheath & Jones claim.

### 5. State the coalescing law precisely

How many transactions a burst becomes depends on when the core drains, which is genuine outside-world timing nondeterminism (the same is true on std). So the semantics could say: the external event sequence is split into contiguous runs, with the split chosen by timing, and each run enters the graph as one transaction carrying the coalesced fold of that run.

For that to be well-defined, the coalescer must be **associative**, because the implementation folds incrementally as events arrive and the result must not depend on grouping. If more than one producer can write the same input (for example nested interrupt priorities), arrival order may be ill-defined too, so either require **commutativity** or restrict each input to a single producer.

This is testable against the oracle: for random event sequences and random partitions into contiguous runs, Bough's output should equal the oracle fed the folded runs.

## Open questions for the Bough session

- **Feature tiers:** core (no allocation) / `alloc` (growable graph without std) / `std` (threads, std sync, ambient thread-locals)? Embedded targets often do have an allocator, so `alloc` is not the same as `std`.
- **Declaring capacity:** const generic, caller-provided `&'static mut` storage, or sized once at init?
- **Dynamic graph restructuring** (not discussed in chat, raised while writing this): Sodium's switch combinators build subgraphs inside transactions. A fixed-capacity graph must bound *peak live* nodes, including switch-created ones, and rely on generational slot reuse for churn. Is switch allowed in bounded mode, and how is its peak bounded? Also consider generation-counter wraparound on a device that runs for days.
- **Coalescing everywhere or only when bounded?** If std mode keeps one-transaction-per-push while bounded mode coalesces, the modes have different observable semantics. Either the oracle models both, or the difference needs to be defined denotationally.
- **Where the coalescing law lives:** it can't be enforced by types, so it needs documenting plus property tests.

## std-leakage audit checklist

- Needs `alloc`: `Vec`, `Box`, `Rc`, `Arc`, `String`, `BTreeMap`.
- Needs `std`: `Mutex`, `RwLock`, `Condvar`, `std::sync::mpsc`, `thread_local!`, `std::thread`, std `HashMap` (`hashbrown` works with `alloc`).
- Atomics: `thumbv7m` (Cortex-M3) has 32-bit atomics but no `AtomicU64`, so check how graph ids and generation counters are generated. `thumbv6m` (Cortex-M0) has no compare-and-swap at all.
- ISR-safe slot writes: the `critical-section` crate (portable) or atomics.
- Ambient context: any per-thread ambient graph or context (like the ambient per-thread `SodiumCtx` pattern in the Sodium Rust work) is std-only. Tokens carrying a graph id plus an explicit context object is the no-std-friendly path.

## Embedded milestone the design should support

The smallest end-to-end test of the claim: a button press flowing through a tiny Bough graph to an LED, on bare metal, no std, on an STM32 Discovery board. The user is learning the embedded toolchain (`memory.x`, `cortex-m-rt`, probe-rs) there first before moving to the Due.

## Suggested skills

- **grilling**: stress-test insights 1 to 5 and the open questions before writing any decision records.
- **domain-modeling**: once grilled, record the resulting decisions and update `CONTEXT.md` in the project's `docs/decisions/` format.
- **engineering:testing-strategy**: design the oracle property tests, especially the partition-into-runs coalescing property.
