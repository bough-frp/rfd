# Strong Separation of Constructing FRP and I/O

One flaw in the Sodium API is that it is possible to use I/O bridge
APIs inside of FRP logic and doing this seriously compromises the
correctness of the FRP engine.

We can do better than this in Rust.

## Flaws in Sodium's API Design

The first problem with the Sodium API is that it is so minimal that
some methods that should only be used for interfacing with the I/O
edge ("World of I/O" in Sodium parlance) are directly on the `Stream`
and `Cell` structs and so they show up in the documentation right next
to all the core FRP combinator primitives. Creating an input FRP
struct requires first creating the input edge struct (an I/O API!), so
from the very jump the Sodium API is _requiring_ the user to mix the
usage of I/O and FRP code.

The second major problem is that the `Transaction` concept is used for
both constructing FRP and in the I/O edge. From an implementation
perspective this makes sense because a transaction is needed. From an
API perspective it again muddles the distinction between the FRP and
I/O. This dual-usage of `Transaction` also doesn't help guide the user
towards the standard pattern of building all FRP up-front and then
"turning the crank" and just sending events to the constructed graph.

Because of these flaws and because Sodium in Rust is a port of the C++
implementation, there is no attempt to leverage unique Rust features
like lifetimes to prevent the use of I/O edge APIs inside FRP because
the design is fundamentally unsuited to doing so.

## Key Changes in API Shape

The key insight is that we can (and should!) actually represent the
difference between "FRP build time" and "I/O event sending time" in
the API, and leverage this to make it impossible to use I/O
constructs inside of FRP core code because the I/O structs don't exist
until _after_ the FRP has been constructed.

A key step in designing a Sodium system is defining the set of events
(`Stream`) and data (`Cell`) that your program takes as input and
produces as output. FRP combinators implement a high-level DSL for
combining snippets of functional code and these combinators need to be
constructed in the same direction as the data flow, so we start with
structs representing our input events and data, and combine them with
combinators to produce structs representing our output events and
data. FRP allows a limited form of cycles in the constructed
graph. These cycles require a forward declaration of a key point in
the loop. The Sodium API to create these loops is functional but quite
awkward to use.

Finally, we should use lifetimes to prevent the user from capturing
the I/O context and storing it to use inside FRP code.

## A Sketch of Using Bough's Prospective Design

In the simple case where there are no cycles, building a Bough system
should look something like this:

```rust
fn main() {
    let rt = bough::Runtime::build(|inputs: bough::Stream<usize>| inputs);

    rt.with_io(|ctx, input, output| {
        ctx.send(input, 0);

        assert_eq!(0, ctx.recv(output));
    });
}
```

When you do have cycles it should look something like this:

```rust
// a simple integer accumulator
fn main() {
    let rt = bough::Runtime::build_with_cycles(
        // Cycle pre-declarations are the second capability argument
        |inputs: bough::Stream<usize>, cycles: bough::Cell<usize>| {
            let sum_events: bough::Stream<usize> =
                inputs.snapshot(cycles, |input, cycle| cycle + input);
            let sum_state = sum_events.hold(0); // Initialize the state to zero
            // Cycle combinator results are the second return value
            (sum_events, sum_state)
        },
    );

    rt.with_io(|ctx, input, output| {
        ctx.send(input, 1);
        assert_eq!(1, ctx.recv(output));

        ctx.send(input, 2);
        assert_eq!(3, ctx.recv(output));
    });
}
```
