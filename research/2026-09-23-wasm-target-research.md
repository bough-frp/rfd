# Bough on the web: what `wasm32-unknown-unknown` + `wasm-bindgen` imposes

_Written 2026-09-23 from the Bough design session. Research, not a record:
nothing here has authority over the RFDs. Design reviewed: RFDs on
`RadicalZephyr:rfd/0002-api-shape` at `780b877`, skeleton
`claude/great-mayer-3pb5gd` at `dfe736b`, plus this week's decisions as
given in the brief. No repository was changed. Scratch crates and logs are
under `scratchpad/wasm/`._

## The answer first

The web target fits the design closely. A single-threaded engine, one
driver, listeners that feed back through an inbox, and a `core::task::Waker`
are exactly what a wasm-bindgen module wants. Four things need attention,
one of them a change.

**New requirements found**

1. Poisoning must work without unwinding. The target is `panic = "abort"`
   on stable, and a panic is a `unreachable` trap: no Rust code after the
   panic hook runs, no drop guard fires, `catch_unwind` cannot catch. The
   instance stays callable afterwards, with std's panic count stuck at one,
   the wasm shadow stack pointer not restored (112 bytes leaked per trapped
   call), every held `Mutex` and `RefCell` borrow stuck. So the graph's
   poison bit can never be set on the way out of a panic. Smallest change:
   make the transaction-in-progress flag the poison indicator. A public
   entry that finds the flag set reports `Poisoned` (it can only mean an
   unfinished transaction), and that entry mirrors the bit into the inbox.
   One flag check per entry stays true; the drop guard that would set a
   separate bit is not needed on any target. RFD-0005 "Panics" and RFD-0006
   "the inbox mirrors the poison bit" need that sentence, plus one saying
   `catch_unwind` around a listener does not exist on abort targets.
2. A rule for DOM closures. `dispatchEvent`, `element.click()` and friends
   run listeners synchronously and nested. A closure holding
   `Rc<RefCell<Graph>>` that calls `graph.send` directly will hit a
   `RefCell` double borrow when a Bough listener dispatches a DOM event
   whose closure sends again. That panic traps; the DOM reports the
   exception and continues, the inner send is lost, and the module runs on
   in a damaged state. The rule: DOM closures send through a `Remote`, and
   a driver future spawned with `spawn_local` pumps when woken. The pump
   runs as a microtask, before the next task and before the next paint.
   Direct sends stay allowed with the hazard stated. `bough-web` defaults
   to the inbox.
3. A wasm CI leg has three concrete requirements. `proptest` must be
   `default-features = false, features = ["std"]` (its `fork` and
   `timeout` features pull `wait-timeout`, which fails to compile on both
   wasm targets). `#[should_panic]` tests are silently ignored by libtest
   on `wasm32-wasip1` (abort target), so the "construction errors panic"
   tests need the `try_` variants or must run under `wasm-bindgen-test`,
   which does support them. `wasm-bindgen-test` needs a CLI of exactly the
   crate's version and `getrandom` configured for `wasm_js`.
4. `critical-section` ships no implementation for wasm. Without the `std`
   feature the two symbols become wasm imports and instantiation fails, not
   linking. Web builds use the `std` feature (a `Cell<bool>` flip on this
   target); a `no_std` web build must call `critical_section::set_impl!`
   with a single-core no-op. State which in the slot feature's docs.

**Decisions affected**

- Poisoning model (RFD-0005, RFD-0006): the change in item 1.
- Test policy (RFD-0001 "The Semantics Are the Test") and the workspace
  manifests: item 3.
- `InputSlot` and the `critical-section` feature: item 4, a sentence.
- `Threaded`, `Remote` and the queue gated on `target_has_atomic = "ptr"`:
  the gate is true on this target with no threads, so all three exist on
  the web. Harmless, one sentence. Note that `JsValue` and every `web_sys`
  handle are `Send + Sync` on a non-atomics build, so `Graph<Threaded>`
  accepts them; `Closure<dyn FnMut(..)>` is not `Send`.
- Size: no decision changes, but the policy should say wasm size is
  reported as information, like trivial-payload overhead. Measured: about
  270 bytes per monomorphized node type after `wasm-opt -Os`, about 400
  before, against about 200 on `thumbv7m` for the same source.

**Decisions unaffected**

Single-threaded engine and single driver. Arena and tokens. Transactions on
the calling thread, listeners after commit with no graph access. Listener
feedback through `Remote`. `Remote` as `Arc<Mutex<queue>>` (the std mutex
is a `Cell<bool>` here; it panics on a recursive lock, so never run user
code under it, which the design already avoids). Entry-point flag check.
`core::task::Waker` as the waker: verified against js-sys's executor from a
DOM closure. `no_std` over `alloc` with `std` default-on: web builds keep
`std`. The thread-id guard: `thread::current().id()` works and is always
`ThreadId(1)`. `Trace for Instant` behind `std`: the type exists, `now()`
traps, and the engine never calls it. Collection policy: linear memory
never shrinks, so the automatic amortized default is the right one for a
long-lived page; nothing to change. Benchmarks native only: there is no
wasm analogue of `iai-callgrind`.

## Facts verified, with toolchain versions

Toolchain: rustc 1.94.1 stable (2026-03-25), cargo 1.94.1, rustup 1.29.0.
Nightly 1.100.0 (2026-09-22) was used for one atomics build only. Node
v22.22.2. Crates from crates.io on 2026-09-23: wasm-bindgen 0.2.128 (crate
and CLI), wasm-bindgen-futures 0.4.78 (now a re-export of
`js_sys::futures`), js-sys and web-sys 0.3.105, wasm-bindgen-test 0.3.78,
wasm-bindgen-rayon 1.3.0 (README only), critical-section 1.2.0, proptest
1.11.0 with rand 0.9.5 and getrandom 0.3.4, send_wrapper 0.6.0,
reactive_graph 0.3.0-beta2, dioxus-core 0.8.0-alpha.1, yew 0.23.0. Tools:
wasmtime 40.0.0 CLI, wasmtime crate 47.0.4 source, binaryen `wasm-opt`
132 and wabt via npm. Std sources: `rust-src` of the stable toolchain.

| Fact | How checked |
|---|---|
| `panic="abort"`, pointer width 32, `target_has_atomic` 8/16/32/64/ptr, default features bulk-memory, multivalue, mutable-globals, nontrapping-fptoint, reference-types, sign-ext | `rustc --print cfg --target wasm32-unknown-unknown` |
| `-C panic=unwind` fails on stable: "the crate `panic_unwind` does not have the panic strategy `unwind`" | compiled `exp1-std-facts` with that flag; same wording in rustc's platform-support page, which says unwinding needs nightly and `-Zbuild-std` |
| A panic is a trap; the instance stays callable; `thread::panicking()` stays true; `set_hook` then panics; shadow stack pointer drops 112 bytes per trapped call and is never restored (wrapped negative after 200,000 traps); a held `Mutex` stays locked; `catch_unwind` traps | `exp1-std-facts` run in Node's WebAssembly API, then through wasm-bindgen's generated glue (`exp2-web/glue.cjs`); std `panicking.rs` `panic_count::increase`, `set_hook` |
| `thread::current().id()` is `ThreadId(1)`; `Builder::spawn` returns `Err(Unsupported)`; `thread::spawn` is `.expect("failed to spawn thread")` | Node run; std `thread/functions.rs:131`; platform-support page line 16 |
| `std::sync::Mutex` is the `no_threads` backend, panics "cannot recursively acquire mutex"; thread locals never run destructors | std `sys/sync/mutex/mod.rs`, `no_threads.rs`, `thread_local/mod.rs` |
| `Instant::now()` and `SystemTime::now()` panic "time not implemented on this platform" | Node run (trap); std `sys/pal/unsupported/time.rs` via `sys/pal/wasm/mod.rs` |
| Linear memory grows from 18 to 531 pages after allocating and freeing 32 MiB and never shrinks; std's allocator is dlmalloc | Node run; `libdlmalloc` in the link line |
| `+atomics` on stable fails to link; `-Z build-std` is rejected on stable; nightly with `-Zbuild-std=std,panic_abort` builds; `thread::spawn` is still `Unsupported` with shared memory | stable and nightly builds of `exp1-std-facts`, the nightly one run in Node with a shared `WebAssembly.Memory` |
| `-Ctarget-feature=+atomics` is unstable and "being phased out; it will become a hard error" | nightly warning, rust-lang/rust#162235 |
| `JsValue` is `Send + Sync` only when atomics is off; `Closure<dyn FnMut(Event)>` is not `Send` | wasm-bindgen `src/lib.rs:178-181`; `exp2-web/examples/sendness.rs` |
| `spawn_local` polls through `queueMicrotask` (fallback: a resolved promise's `then`); the waker is `Rc`-based | js-sys `src/futures/queue.rs`, `task/singlethread.rs` |
| A DOM closure can wake the driver task; the poll runs before a `setTimeout(0)` | `exp2-web/tests/web.rs` under `wasm-bindgen-test-runner` in Node: order `poll, closure, after dispatch, poll, timeout` |
| `dispatchEvent` is synchronous and nests; listener exceptions are reported, not rethrown | MDN wording; Node's WHATWG `EventTarget` (`reentrant.mjs`); `tests/web.rs` where `dispatch_event` returned `Ok(true)` after the inner closure trapped |
| Same-closure re-entry throws "closure invoked recursively or after being dropped" without a Rust panic | wasm-bindgen `convert/closures.rs:92`; test in `tests/web.rs` |
| Microtask checkpoint when the JS execution context stack empties; `setTimeout` clamped to 4 ms above nesting level 5; rAF callbacks run in "update the rendering" and hidden documents are filtered out | HTML spec pages fetched and grepped |
| Latency: `queueMicrotask` 1 µs median, `MessageChannel` 2 µs, `setTimeout(0)` 1.1 ms (Node floor), 10 nested timeouts 11.9 ms | `latency.mjs` in Node |
| `Leaf<JsValue>`, `Leaf<Element>`, `Leaf<Closure<..>>` in cells and an `Element` captured by a listener compile in `Local` mode against the skeleton; `Graph<Threaded>` also accepts `Leaf<JsValue>` | `cargo check --target wasm32-unknown-unknown` on `exp2-web` |
| A dropped `Closure` invalidates its JS function; later calls throw | wasm-bindgen `closure.rs` docs at lines 393-397 |
| Per node type: 397 bytes raw, 272 after `wasm-opt -Os` on wasm32; 204 on `thumbv7m-none-eabi` | `exp4-size`, 10/110/210 generated node types, `opt-level = "s"`, lto, one codegen unit |
| wasip1 tests run under wasmtime; `should_panic` tests are ignored there; proptest default features fail on both wasm targets | `exp3-tests` |
| `wasm-bindgen-test` in Node runs proptest and `should_panic` tests; needs `getrandom` `wasm_js` feature plus `--cfg getrandom_backend="wasm_js"`; the CLI checks the crate's schema version | `exp3-tests`; cli-support `lib.rs:453-459` |
| GitHub `ubuntu-24.04` image ships Chrome 152 with ChromeDriver 152 and Firefox 155 with geckodriver 0.37.1; no wasmtime | runner-images README |
| `critical-section` without `std` leaves `env._critical_section_1_0_acquire/release` as imports; instantiation fails; with `std` it is a std mutex plus a thread-local re-entrancy flag | `exp5-cs`, `wasm-objdump`, Node |
| `wee_alloc` is unmaintained (RUSTSEC-2022-0054) | advisory-db |
| wasmtime fuel: CLI `-W fuel=N`; crate `Store::set_fuel`/`get_fuel` | `wasmtime run -W help`; wasmtime 47.0.4 `store.rs` |
| `web_sys::Performance::now`, reachable from `Window` and `WorkerGlobalScope` | web-sys generated sources |
| Leptos wraps browser-only values in `SendWrapper`; Dioxus `spawn` takes `'static` non-`Send` futures; Yew uses `spawn_local` | crate sources |

## The ten questions

### 1. Target facts

`panic = "abort"` is the only strategy on stable; the shipped std has no
`panic_unwind`. A panic runs the panic hook (this is where
`console_error_panic_hook` prints), then executes `unreachable`, which
surfaces in JS as `RuntimeError: unreachable` thrown out of the export. The
module is not dead: every export stays callable, with the wasm-bindgen glue
too (`alive_after` returned 42 after two traps). It is damaged: the panic
count is stuck, so `thread::panicking()` is true forever and `set_hook`
panics; the shadow stack pointer is not restored, so each trapped call
leaks its frames; a `Mutex` held at the trap stays locked and a `RefCell`
borrowed at the trap stays borrowed. wasm-bindgen 0.2.128's `handler.rs`
says the same in its words: a hard abort poisons the instance. Its
`set_on_abort` works only on unwind builds; its `schedule_reinit` creates a
fresh instance on the next export call and works on abort builds, which is
the only recovery, and it loses every Rust value.

Consequence for Bough: poisoning cannot be set by a guard; the in-progress
flag must be the poison, as the answer says. `catch_unwind` is unavailable,
so "a user who wants a listener to survive its own panic wraps it in
`catch_unwind`" does not apply on this target. The web adapter learns of a
poisoned graph at the next `pump`, from the driver future, and can tear
down once instead of trapping on every event.

`std::thread::current().id()` works, is always `ThreadId(1)`, and costs a
thread-local read. `std::thread::spawn` panics; `Builder::spawn` returns
`Err(Unsupported)`. `std::sync::Mutex` is a `Cell<bool>` that panics on a
recursive lock. `Instant::now()` and `SystemTime::now()` panic; the types
exist, so `impl Trace for Instant` compiles. Thread-local destructors never
run. The default allocator is dlmalloc.

### 2. Threads on the web

A stable build cannot share a `Graph` across workers. Without the `atomics`
feature there is no shared memory; each worker instantiates its own module
with its own linear memory, and `postMessage` copies bytes. Threads need
nightly, `-Zbuild-std=std,panic_abort`,
`-Ctarget-feature=+atomics,+bulk-memory`, `--shared-memory` at link, and
COOP/COEP headers so the page may use `SharedArrayBuffer`
(wasm-bindgen-rayon README, which pins `nightly-2025-11-15`). Even then
`std::thread::spawn` is `Unsupported`; workers are spawned by JS glue such
as wasm-bindgen-rayon's, and the nightly warning says the
`-Ctarget-feature` route itself is being phased out.

Position for Bough: on stable, the web is one thread. `Local` is the web
mode. `Threaded` compiles because `target_has_atomic = "ptr"` holds, and it
even accepts `JsValue`, but it buys nothing there. `Remote` works on that
one thread and is the recommended path for every DOM callback (question
4). A worker feeds the graph by `postMessage` to the main thread, where a JS
handler calls an export that does `remote.send`. Shared-memory threads
with a `Graph<Threaded>` in one worker and `Remote`s in others would fit
the design as it stands, and are out of scope until the toolchain is
stable. One caution for that future: on a browser main thread a blocking
wait is not allowed, so the inbox lock must stay uncontended-short, which
it is; this was not verified here.

### 3. The driver on the web

`wasm_bindgen_futures::spawn_local(driver)` where `driver` is a future
owning the `Graph<Local>`: on each poll it stores `cx.waker().clone()`
into the graph (or the inbox stores it through `set_waker`), calls
`pump()`, and returns `Pending`. `Remote::send` from a DOM closure
enqueues and calls `wake_by_ref`; js-sys's executor schedules the poll
with `queueMicrotask`. Verified order from a synchronously dispatched DOM
event: `poll, closure, after dispatch, poll, timeout`. The `Waker` is
`core::task::Waker`, `Send + Sync` by type, `Rc`-based inside, and usable
from any `Closure` on this thread. `spawn_local` takes a `'static`,
non-`Send` future, so the `Local` graph moves in.

Pump points: a microtask runs when the JS stack empties, so after each
native event listener returns and before the next task or paint; latency
is microseconds. `setTimeout(0)` is a task, 1 ms in Node and clamped to
4 ms in browsers once nested five deep. `requestAnimationFrame` runs once
per rendering opportunity, about 16.7 ms at 60 Hz, and not at all while
the document is hidden. The pump is the microtask through `spawn_local`.
rAF belongs in a listener that schedules a paint, never under the pump.

### 4. Re-entrancy from DOM callbacks

Synchronous re-entrant dispatch is real: MDN says `dispatchEvent` "invokes
all applicable event handlers synchronously before returning", and Node's
WHATWG `EventTarget` shows `outer:start, inner:ran, outer:end`. Two cases:

- The same closure re-entered: wasm-bindgen zeroes the closure's state
  pointer during a call, and the nested call throws "closure invoked
  recursively or after being dropped". No Rust panic; the nested send is
  lost; the error is reported by the dispatcher.
- A different closure sharing `Rc<RefCell<Graph>>`: `borrow_mut` panics
  "already borrowed", which traps. The DOM dispatcher reports the
  exception and continues; `dispatch_event` returned `Ok(true)` to the
  outer listener, the outer transaction finished, the inner send was lost,
  and `thread::panicking()` was true from then on. If the trap is inside
  the outer transaction instead (a user function panicking), the borrow
  is never released and every later direct send traps.

Rule to document: sends from DOM closures go through the inbox; the driver
task pumps. This is the design's own rule for listeners ("a later
transaction, never a nested one") applied to the DOM, and it costs one
boxed unit and one microtask. Direct `graph.send` from a closure is
allowed only when no listener can reach the DOM synchronously, which no
check can establish, so `bough-web` defaults to the inbox and exposes the
graph only to the driver.

### 5. JS values in the graph

Verified in `Local` mode against the skeleton: a hold of `Leaf<JsValue>`,
a constant `Leaf<web_sys::Element>`, a constant
`Leaf<Closure<dyn FnMut(web_sys::Event)>>`, a stream listener capturing an
`Element`, and `listen_cell` on `Leaf<JsValue>`. The premise that these
are not `Send` is false on a plain build: `JsValue` and every handle over
it are `Send + Sync` whenever `atomics` is off, on native too. Only
`Closure<dyn FnMut(..)>` fails `Send`, through its `dyn FnMut`. So
`Threaded` accepts DOM handles on the web and rejects closures. Nothing to
change; a sentence in the mode docs.

`Drop` of a `Closure` inside a collected node: dropping invalidates the JS
function, and a DOM listener still registered with it throws on the next
event. Collection runs between transactions on the driver thread, so
calling into JS from `Drop` is fine. The adapter should store a guard that
removes the listener on drop, not a bare `Closure`. wasm-bindgen 0.2.128
also has `ScopedClosure<'a, T>`; Bough's `'static` listeners use
`Closure<T>`, its `'static` alias.

### 6. Memory and collection

Linear memory only grows, verified: 32 MiB allocated and freed left 531
pages mapped. So the high-water mark of the page's life is what the tab
pays. The arena's slot reuse and generation bump already keep node churn
inside the mark; the collector should keep its mark stack and work
vectors allocated between runs rather than allocate per collection. The
automatic amortized policy is right for a long-lived page; manual is for
frame loops; after-every-transaction stays a test setting. `wee_alloc` is
unmaintained with known leaks; std's dlmalloc is the default and the
answer. Nothing to change.

### 7. Size

Measured with a stand-in engine (an arena of boxed `dyn Node`, each a
monomorphized `Map<F>` with a distinct closure, `opt-level = "s"`, lto,
one codegen unit): 10, 110 and 210 node types gave 6,002, 45,736 and
85,709 bytes, about 397 bytes per node type; after `wasm-opt -Os` 3,901,
30,835 and 58,333, about 272 bytes. The same source on
`thumbv7m-none-eabi` grew about 204 bytes per type. Each type carries its
closure, its `evaluate`, its vtable and a monomorphized `add`. The
fixed cost of a wasm-bindgen module against the skeleton (with `todo!()`
bodies) was 29 KB, 16.7 KB after `wasm-opt`. A thousand distinct node
closures cost about 300 KB. No decision changes: fusion already minimizes
node count, and a web app of that size is not size-bound by Bough. State
size as reported information, not a bar.

### 8. Testing and CI

Two routes, both verified:

- `wasm32-wasip1` under wasmtime: `cargo test --target wasm32-wasip1` with
  `runner = "wasmtime run --dir=."`. The oracle and engine tests run as
  they are, with proptest at `default-features = false, features =
  ["std"]`. `#[should_panic]` tests are ignored by libtest on this abort
  target. wasmtime is not on the GitHub runner; use
  `bytecodealliance/actions/wasmtime/setup` or download a release.
- `wasm32-unknown-unknown` under `wasm-bindgen-test`: `cargo install
  wasm-bindgen-cli` at the crate's exact version (the CLI checks a schema
  version), `runner = "wasm-bindgen-test-runner"`, `getrandom` with the
  `wasm_js` feature and `--cfg getrandom_backend="wasm_js"` in rustflags.
  Node is the default and needs nothing else; `#[should_panic]` works.
  Browsers with `WASM_BINDGEN_USE_BROWSER=1`; the runner image has Chrome
  and Firefox with matching drivers.

What to put in CI now: `cargo check --target wasm32-unknown-unknown -p
bough`, later `--no-default-features`. When the engine lands: the wasip1
leg for `bough` and `bough-oracle`, and the Node leg for `bough-web` only.
`iai-callgrind` has no wasm analogue. wasmtime fuel is a deterministic
count through the embedding API, but it counts Cranelift's code on wasip1,
not V8's, so it is not a regression gate for the web. A web measurement is
`performance.now()` around a batch of transactions, through
`web_sys::Performance`, run by hand and reported as information.

### 9. The `critical-section` feature and slots on the web

An `InputSlot` as a `static` with a critical section is fine on a
single-threaded module. `critical-section` 1.2.0 provides no
implementation for wasm: without one, the acquire and release symbols
become module imports, because the target links with `--allow-undefined`,
and the failure is at instantiation ("Import #0 module=env"). With the
`std` feature the implementation is a std mutex plus a thread-local
re-entrancy flag, which on this target is a `Cell<bool>` flip. Web builds
should use the `std` path, which is the default; a `no_std` web build
must register a single-core no-op with `set_impl!`. Say so in the
feature's documentation.

### 10. Anything else

- JS exceptions never trap Rust: `js_sys::Function::call1` returns `Err`
  on a throw, so a `Function` used as a listener callback cannot poison the
  graph from JS; only Rust panics can.
- Wakers: `core::task::Waker` is `Send + Sync` by type; js-sys's is an
  `Rc` that is only safe on one thread. On a non-atomics build there is one
  thread, so `set_waker` and the inbox's stored waker are sound.
- Frameworks as hosts: Leptos requires `Send + Sync` for reactive values
  and wraps browser-only values in `SendWrapper`; a `Graph<Local>` in a
  Leptos signal goes in `SendWrapper` too. Dioxus `spawn` and Yew's
  `spawn_local` take `'static` non-`Send` futures. All three are
  single-threaded on the web, none owns a thread, and a Bough driver
  future is a spawned local task in each. This is the Bevy shape: the host
  schedules the driver, the graph belongs to the driver.
- The instant rule: each DOM event's unit is one transaction, and units
  are never merged, so nothing batches a frame into one instant unless an
  adapter uses `Remote::transaction` on purpose. The trap is pumping on
  rAF: it delays every transaction to the next frame and stops in a hidden
  tab. The pump is the microtask.
- Default target features rose with Rust 1.87 (bulk-memory,
  nontrapping-fptoint); engines older than that are out, which is fine.
- `console_error_panic_hook` runs before the trap and is the only way to
  see a panic message; the adapter should install it.

## Bough's decisions against the target

| Decision | Verdict |
|---|---|
| Single-threaded engine, single driver | Unaffected; it is the web's model |
| `Local` stores non-`Send`; `Threaded` requires `Send`, `Graph<Threaded>: Send` | Needs a sentence: `JsValue` is `Send` without atomics, `Closure` is not; `Local` is the web mode |
| `Threaded`, `Remote`, unit queue only where `target_has_atomic = "ptr"` | Needs a sentence: the gate is true here, so all three exist on the web |
| Arena, tokens of index, generation, graph id | Unaffected |
| Transactions run inside `send`, `transaction`, `pump` on the calling thread | Unaffected |
| Listeners after commit, no graph access, feedback through `Remote` | Unaffected; it is the DOM rule too |
| `Remote` = `Arc<Mutex<queue>>`, `Send + Clone` | Unaffected; the mutex is a `Cell<bool>`; keep user code out of the lock |
| Entry-point flag check against a smuggled `Graph` | Unaffected |
| Panic escaping a transaction poisons; no rollback; `catch_unwind` for listeners | Needs a change: the in-progress flag is the poison, an entry that finds it set reports `Poisoned` and mirrors it to the inbox; `catch_unwind` does not exist here |
| Core `#![no_std]` over `alloc`, `std` default-on | Unaffected; web builds keep `std` |
| Waker is `core::task::Waker`; driver future stores `cx.waker()` and pumps | Unaffected; verified with js-sys's executor |
| `InputSlot` static, `critical-section` behind a feature, std mutex under `std` | Needs a sentence: `std` path on the web; `set_impl!` for `no_std` web |
| `Remote` guard by thread id under `std` | Unaffected; always `ThreadId(1)` |
| `Trace` for `HashMap`, `HashSet`, `Instant` behind `std` | Unaffected; `Instant::now()` traps, the engine never calls it, adapters use `performance.now()` |
| Benchmarks native only; no timers in the engine | Unaffected; add a sentence that wasm size is reported as information |
| Oracle port, property tests, seeded shuffling | Needs a change: proptest without default features; `should_panic` tests use `try_` variants or the Node leg; a wasm CI leg |
| Collection: automatic amortized, manual, after-every for tests | Unaffected; a sentence that memory never shrinks |

## What I did not do

- No run in a real browser: the container's Chromium 141 does not match
  its ChromeDriver 147. Browser facts come from the HTML and DOM specs,
  MDN, and Node's WHATWG `EventTarget` and `queueMicrotask`.
- No measurement of the real engine: the skeleton's bodies are `todo!()`.
  The size figures use a stand-in arena of boxed nodes, and the earlier
  Cortex-M figure was measured on different code.
- No `panic = "unwind"` build on nightly, and no wasm-bindgen-rayon build;
  the nightly work stopped at a `-Zbuild-std` atomics build of the std
  facts crate, run in Node with shared memory.
- No compile against Leptos, Dioxus or Yew; their sources were read for
  the spawn bounds and `SendWrapper` only.
- Not verified: the behaviour of a blocking wait on a browser main thread
  under shared memory, and the browser's own coalescing of pointer events.
- rAF latency was not measured; the spec's rendering-opportunity rule is
  cited instead.
