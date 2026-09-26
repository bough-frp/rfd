# Slot churn simulation

_2026-09-23. An idea, not research: nothing here has been tried._

## The thought

RFD-0007 retires an arena slot whose generation reaches its maximum, so a
stale token from before a wrap can never validate. On a device that runs
for months, retirement turns the generation counter into a budget: every
node freed and re-made spends one, and a graph that churns one slot at a
kilohertz spends the budget in fifty days, while a first-in first-out free
list spreads the same churn across every slot. The number an embedded
designer needs is not the mechanism but the uptime it implies for their
program, and today nothing tells them.

## What we might build

A simulation harness, behind a cargo feature, that runs a `no_std` Bough
project on the development host with no cross-compilation. The user
describes the expected rate of each input slot they declared, the harness
drives the slots at those rates faster than real time, and it measures
what the graph does: generations consumed per arena slot, the high-water
mark of live nodes, and, once the bounded engine exists, how close the
program comes to exhaustion. From the churn it estimates the uptime until
the first retirement.

Faster than real time is cheap here. The engine has no timers, so time is
the order of events; a simulated hour is a number of pumps, not an hour.

## Where the shape comes from

Datomic's Simulant: a simulation is a model of the actors and their
schedules, the run is data, and every result is recorded so that it can be
queried after the fact rather than asserted during the run. A slot's rate
model is an actor; a run is the schedule it produced; the churn and the
high-water marks are the recorded results.

## What it would take to find out

- How a rate is described: a constant, a distribution, a trace recorded
  from the real device.
- What the harness counts, and whether the `statistics` feature already
  carries enough of it.
- Whether it is a feature of the engine crate or a `bough-sim` crate
  beside the adapters.
- Whether the estimate is worth trusting: run it against a real device for
  a day and compare the churn.
