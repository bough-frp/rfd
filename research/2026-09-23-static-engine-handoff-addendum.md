# Addendum: a second compile target and a third engine shape

_Written 2026-09-23 in a claude.ai conversation with Zefira. Extends
`HANDOFF-static-engine.md`: its goal, constraints, voice, working style and
deliverable all stay in force. Transient working state, not a record:
nothing here has authority over the RFDs._

Two additions to your exploration. Each ends in something your report has
to contain.

## 1. Compile every experiment for `thumbv6m-none-eabi` too

Question 5 fixes `thumbv7m-none-eabi` as the target. Add
`thumbv6m-none-eabi` (Cortex-M0 and M0+) as a second target for every
compile experiment. It needs no hardware: `rustup target add
thumbv6m-none-eabi`.

The M0 is the floor on atomics, not on allocation. Its rustc target page
lists `core` and `alloc`, and the target has no compare-and-swap: atomics
are 32-bit load and store only. The handoff's verified stub already fails
there on `alloc::sync` and `AtomicU32::fetch_add`. A second column shows
Zefira which engine shape reaches the M0 without `portable-atomic`, which is
part of what each Q1 option buys.

Reference RAM budgets for question 10: the Due's SAM3X8E has 96 KB of SRAM.
The M0 part Zefira would buy, the STM32G071RB on a NUCLEO-G071RB, has 36 KB.

**Done when:** every experiment in the report records its result on both
targets, and every `thumbv6m` failure names the item that failed.

## 2. Evaluate a third shape: the bounded dynamic engine

### The premise to test

The handoff's section "Why a static engine is a different engine" rests on
one step: heterogeneous nodes in one arena need type erasure, and erasure is
`Box<dyn Fn>`. Erasure needs indirection, not a heap. A trait object can
point into storage the caller provides. If that holds without `alloc`, there
is a Q1 option between (a) and (b): keep the dynamic engine and give it a
bounded storage backend. Call it option (d), the **bounded dynamic engine**.
That name is a working label for this report, not glossary vocabulary.

Nobody has compiled this yet. Compile the claim before building on it.

### Experiment E1: bump region

Build-time node state is written into caller-provided storage, a
`&'g mut [MaybeUninit<u8>]`, and the node table holds `&'g mut dyn`
references into it. This is the smallest proof that erasure works in
`core`. It reclaims nothing, so it models a graph with no `construct`. It is
also the no-waste baseline E2 is measured against.

### Experiment E2: fixed-size erased slot

Each arena slot holds a node's whole state inline: fused closure, held
value, memo. The slot is N bytes at a fixed alignment. Beside the bytes it
stores one monomorphized function pointer that casts the slot back to
`*mut dyn Node`, so the compiler still supplies the vtable and
`drop_in_place` works through the same pointer. An inline `const` assertion
rejects any node type larger than N or more aligned than the slot. Uniform
slots make reclamation a free list, so the arena, generations, `Trace`,
`construct` and the switches keep their current design.

Write down the unsafe invariants. Run the slot's tests under Miri on the
host if Miri is available.

### What to find out

- **Allocation inventory.** Every allocation site in the skeleton at
  `dfe736b` gets a bounded replacement or is named as unsolved. Known sites
  from the handoff: the `Arc` waker, the `fetch_add` graph-id counter, the
  inbox's `Mutex` and `Box<dyn FnOnce + Send>`, and `Trace`'s `HashMap` and
  `HashSet`. Expect more: dependents lists, listener lists, the
  per-transaction mark and order buffers, the `split` and `defer` child
  queue.
- **Slot exhaustion.** `construct` can now run out of slots. The semantics'
  `Execute` always succeeds, so this is a new failure mode. State the policy
  it needs, and whether that policy is a deviation needing its own RFD, the
  same way a static-tier subset would be.
- **Costs.** RAM waste on the flagship example (E2's slots against E1's
  exact bytes), against both reference budgets. Lines of unsafe code. The
  exact compiler error a user sees when a closure exceeds N: inline `const`
  assertions fail after monomorphization, so judge its legibility the way
  question 10 judges the static engine's errors.
- **Map onto the handoff's questions.** A hypothesis to confirm or break:
  questions 1, 2 and 9 dissolve, because the surface and the random-program
  oracle test stay as they are. Questions 3 and 4 resolve to "kept",
  because `construct` draws a slot and collection frees one, so RFD-0003
  survives apart from slot exhaustion. Questions 6, 7 and 8 carry over with
  the same fixed-capacity answers the static tier needs. Question 5 becomes
  the current transaction over pre-sized buffers. For question 11, say
  whether a storage seam is a narrower interface than a whole-engine
  backend.
- **Prior art for question 12.** Inline trait-object storage exists as
  crates; `smallbox` and `stack_dst` are leads to verify. The zero-dependency
  rule leaves any of them as a question for Zefira.

**Done when:** E1 and E2 are in the report as source, output and toolchain
version on both targets; the allocation inventory covers every site; and
the Q1 recommendation chooses among (a), (b) and (d), with (d)'s costs
measured.
