# The I/O Edge: Remotes, Drivers, and Threading Modes

**State:** discussion · **Authors:** Zefira Shannon, Claude

The engine is single-threaded and stays so. The requirement is that it
be usable from any threaded or async runtime with minimal friction,
from several at once, and without the library choosing a channel
implementation on its users' behalf. Interrupt handlers and DOM
callbacks are two more callers; [RFD 7](./rfd-0007-targets.md) gives
the first an input slot and the second a rule, and this RFD keeps the
queue.

## The Driver and the Remote

A single-threaded engine needs a single driver: whoever owns the
`Graph` runs its transactions. Making users hand-route every I/O
source into that driver through channels of their own is the wrong
place to put the friction. Making each integration crate own the graph
fixes that and creates a worse problem: two integrations cannot share
one graph. So the runtime-agnostic part of the glue lives in the core.

`graph.remote()` returns a `Remote`, which is `Send + Clone`, backed
by an inbox behind a standard-library mutex and a waker, a
`core::task::Waker` registered through `graph.set_waker`.
`remote.send(input, value)` locks, pushes a unit holding one boxed
send, unlocks, and wakes. It is synchronous and never blocks on the
graph. `remote.transaction(|tx| ...)` pushes a unit holding a
`Send + 'static` closure that runs on the driver with a `Transaction`,
so its sends are simultaneous. The driver calls `graph.pump()`, which
runs each pending unit as one transaction in arrival order, and
arrival order is the total order the semantics need. A unit is the
unit of simultaneity: it is never split and two are never merged,
which is what lets a double send inside one be reported at `pump`
([RFD 5](./rfd-0005-transaction-protocol.md)). Transported values
must be `Send`; nothing else changes, and a `Remote` works with a
`Graph<Local>`. `Remote` is an `Arc`, so it exists where the target
has pointer atomics ([RFD 7](./rfd-0007-targets.md)).

The waker is the standard library's own. A driver that is a future
stores `cx.waker().clone()` on each poll, pumps, and returns pending,
so a tokio, smol or embassy task drives the graph with no channel in
between; a thread driver builds a waker from an `Arc` through
`alloc::task::Wake` and blocks on whatever that wakes; a bare-metal
main loop that sleeps on the interrupt itself gives `Waker::noop()`.
`core::task::Waker` needs no allocator and no atomics, so `set_waker`
exists on every target. A function pointer in an `Arc` was the first
draft; it was the same thing under a name every runtime would have to
learn. Latency is the distance from a send to the next pump, a property
of where the driver sits: a woken thread pumps at once, a future at its
next poll, a frame-based host wherever its embedder placed the runner,
and an adapter states its placement and the latency it implies.

The core ships one driver: a dedicated standard-library thread that
builds the graph, hands back the edge and a `Remote`, and blocks on a
condition variable until woken. An integration crate is then a few
dozen lines: a waker for its runtime, a driver loop if it wants one,
and adapters from `listen` into its channel types, which is where
queues live; the core has no queue type
([RFD 2](./rfd-0002-strong-io-separation.md)). Multiple
integrations coexist because none of them owns anything; they all
hold `Remote`s. A GUI owning the graph on its main thread works with
tokio network tasks holding `Remote`s at the same time, and the GUI's
non-`Send` state stays in listener closures on the driver thread. No
adapter crate ships with the first version; `bough-tokio` is the first
afterwards, once the performance bar has been measured.

## The Chat Room

One input takes a username and a line; another takes a username and
the channel that reaches that user's socket. Each per-user tokio task
holds a `Remote` clone, registers its channel when the connection
opens, and sends each line it reads. The routing table is a cell in
the graph, so the one outbound listener captures nothing that changes,
and it is attached before the graph moves into the task that pumps it,
since `listen` needs `&mut Graph`. The user writes no channel plumbing
for inputs; the only channels in sight are the per-socket ones tokio
needs anyway.

```rust
type User = String;

#[derive(Trace)]
struct Members {
    #[trace(skip)]
    by_user: HashMap<User, mpsc::Sender<String>>,
}

async fn serve(listener: TcpListener) -> std::io::Result<()> {
    let (mut graph, (joins, messages, outbound)) = Graph::build_threaded(|b| {
        let (joins, joins_in) = b.input::<(User, mpsc::Sender<String>)>();
        let (messages, messages_in) = b.input::<(User, String)>();
        let members = joins.accumulate_mut(
            b,
            Members { by_user: HashMap::new() },
            |(user, sender), m| {
                m.by_user.insert(user, sender);
            },
        );
        let outbound = messages
            .snapshot(members, |(user, line), m| {
                let recipients: Vec<_> = m.by_user.values().cloned().collect();
                (recipients, format!("{user}: {line}"))
            })
            .node(b);
        (joins_in, messages_in, outbound)
    });

    graph
        .listen(outbound, |(recipients, text)| {
            for sender in recipients {
                let _ = sender.try_send(text.clone());
            }
        })
        .keep();

    let remote = graph.remote();
    tokio::spawn(std::future::poll_fn(move |cx| {
        graph.set_waker(cx.waker().clone());
        graph.pump();
        std::task::Poll::<()>::Pending
    }));

    loop {
        let (socket, address) = listener.accept().await?;
        let remote = remote.clone();
        tokio::spawn(async move {
            let user = address.to_string();
            let (reader, mut writer) = socket.into_split();
            let mut lines = BufReader::new(reader).lines();
            let (sender, mut inbox) = mpsc::channel::<String>(16);
            remote.send(joins, (user.clone(), sender));
            loop {
                tokio::select! {
                    line = lines.next_line() => match line {
                        Ok(Some(line)) => remote.send(messages, (user.clone(), line)),
                        _ => break,
                    },
                    Some(text) = inbox.recv() => {
                        if writer.write_all(text.as_bytes()).await.is_err() {
                            break;
                        }
                    }
                }
            }
        });
    }
}
```

The example compiles against the API skeleton, with `#[derive(Trace)]`
expanded by hand until the derive lands. The driver is a future: each
poll stores the task's waker, pumps, and returns pending, and every
`Remote::send` wakes it; there is no channel and no notifier. An
earlier sketch attached
the outbound listener after the spawn and captured the per-user
senders in it. That does not compile, because `graph` has moved, and
making it compile would have needed a shared map of senders behind a
mutex, which is the plumbing this design exists to remove; a routing
table that is a cell is the FRP answer, and it is what "anything a
listener needs is snapshotted into the graph" means in practice.

## The New Hole in the Wall, and Its Guard

`Remote` is `Send + Clone + 'static`, so a `map` closure can capture
one and send from inside graph code. It is not reentrant, since a
send only enqueues, but it is I/O inside FRP logic. The inbox records
the driver's thread id and a flag that the driver sets when a
transaction begins and clears before its listeners run, so the flag
covers evaluation and commit alike, `accumulate_mut` closures
included. It is not the graph's own flag, which stays set until the
transaction has finished and is the poison
([RFD 5](./rfd-0005-transaction-protocol.md)). A `Remote::send` from
the driver thread while the inbox's flag is
set is an error in both build modes, `InsideTransaction` from
`try_send`, since it is a logic error with an observable outcome and
not an unobservable one
([RFD 5](./rfd-0005-transaction-protocol.md)). A remote send whose
input was collected before the driver pumps is discovered at `pump`
and follows the debug/release rule for unobservable operations. From
another thread it enqueues. From a listener it enqueues too, since
the flag is clear by then, which is the sanctioned way for I/O to feed
back into the graph: a later transaction, never a nested one. Leaving
it unguarded and documented was the alternative; a check on a path
that is already a bug costs nothing. The thread id exists under `std`;
on bare metal the guard is documented and unchecked, since the
interrupt path is the input slot and a `Remote::send` from a handler is
misuse ([RFD 7](./rfd-0007-targets.md)).

The inbox also mirrors the graph's poison. Once an entry has found the
graph poisoned ([RFD 5](./rfd-0005-transaction-protocol.md)), it sets
the bit in the inbox, so `Remote::send` panics and `try_send` returns
`Poisoned` from then on and no thread keeps filling an inbox that no
pump will ever drain; without the mirror, `try_pump` would fail forever
while every remote kept getting `Ok`, and the inbox would grow without
bound behind it.

## Threading Modes

`Remote` removes most of the pressure, but a non-`Send` graph still
has to be built on the thread that runs it, so a tokio-native user
cannot hold it in a spawned task or behind an `Arc<Mutex>`. The graph
and the build context take a mode parameter, declared as
`struct Graph<Mode = Local>` and `struct Build<Mode = Local>`. In
`Threaded` mode every value and closure the graph stores must be
`Send`, checked once per materialization and at `listen` through a
per-mode `Accepts<T>` trait, and `Graph<Threaded>` is `Send`. `Build`
carries the mode because materializers see only `Build`, so the check
has nowhere else to attach: a threaded graph's build closure receives
a `Build<Threaded>`, and an `Rc` captured in a closure there is a
compile error, which a stub confirmed. Single-threaded users never see
either parameter, and their `Rc<RefCell<UiState>>` captures keep
compiling; a helper that builds graph for either mode is written over
`Build<M>`. `Graph::build` builds a `Local` graph and
`Graph::build_threaded` a `Threaded` one, two constructors rather than
one, because a defaulted type parameter takes no part in inferring an
associated function: `Graph::build(|b| ...)` with a generic `build` is
"type annotations needed", and two inherent `build`s are ambiguous,
as a stub of the API confirmed. Tokens are plain integers and `Send`
in every mode. `Remote` does not carry the mode; `Listener` and
`Anchor` do, as `Listener<M = Local>` and `Anchor<M = Local>`, because
the flag a handle shares with its node is a counted cell in `Local` and
an atomic in `Threaded`, and the parameter is defaulted so `Local` code
never writes it. `Threaded` itself exists only where the target has
pointer atomics ([RFD 7](./rfd-0007-targets.md)).

Requiring `Send` everywhere would have killed the UI case, where
toolkit handles are not `Send`. A non-`Send`-only graph would have
left tokio users with the dedicated-thread driver as the sole option,
which is friction on the first line. Retrofitting a type parameter
touches every signature, which is why this is decided now rather than
later.
