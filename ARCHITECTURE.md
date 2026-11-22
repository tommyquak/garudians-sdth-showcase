# Architecture

How the submitted version fits together.

```mermaid
flowchart LR
    op["Operator page<br/>3D view, threat tier,<br/>peacetime approvals"]

    subgraph sim["Simulator"]
        direction TB
        world["World<br/>5,710 buildings, collateral map"]
        raid["Raid<br/>12 to 48 attackers, 1 to 8 axes or trucks"]
        phys["Flight model and collisions"]
    end

    subgraph drone["Each defending drone (onboard)"]
        direction TB
        sense["Own sensors<br/>seeker, relay, site"]
        model["Own world model<br/>tracks, reports carried with their age"]
        mesh["Signed mesh<br/>Ed25519 keys, heartbeats, friend or foe"]
        alloc["Deterministic allocator<br/>same bids, same answer"]
        act["Launch, aim, kill point"]
        sense --> model
        mesh <--> model
        model --> alloc --> act
    end

    engine["Rust engine<br/>routes between buildings,<br/>line of sight, raid planning"]

    op -- "tier, approvals" --> sim
    sim -- "what each sensor can see" --> sense
    act -- "commands to its own airframe" --> phys
    act <--> engine
    sim <--> engine
```

## The rule that shapes everything

A drone decides only from what it senses or is told. No decision may read the simulator's true state:
true positions, true identities, which drones are destroyed, or the raid roster. A scanner lists every
place the code reads true state and files it as display, measurement or decision. The test suite fails
if a change adds a decision read.

Unsigned messages are bounded: they can never override what a drone's own seeker holds, carry a time in
the future, or permanently remove a resource.

## Allocation without a leader

Each drone holds its own picture and the signed bids it has heard. It runs the same deterministic
allocator over them. Targets are never ordered by a private name, so two drones holding the same bids
reach the same assignment whatever they call the targets.

## The mesh

Every aircraft has an Ed25519 key. Friend or foe is a signed challenge and response with per-link
session keys. Each drone sends signed heartbeats, and a peer that falls silent is marked lost. Drones
judge who they can reach from their own record of who they have heard, not from a global view of the
radio links.

## Collateral

The collateral map is pre-loaded map data: estimated people exposed per 10 m cell, by day and by night,
with schools, hospitals, clinics, eldercare, childcare and places of worship weighted higher. A net
interceptor scores kill points within 3 s of its earliest possible intercept by where the debris is
predicted to fall, and picks the lowest. It never delays a kill for longer than that.

## Testing and measurement

- Runs are deterministic: the same seed and code give a bit-identical run.
- Each change to decision code was gated by the full test suite, a weakening harness (every decision
  rule is deliberately broken one at a time, and a test must catch each break), and the ground-truth
  scanner.
- Behaviour changes were measured, not asserted: paired runs over 8 or more distinct raids, reported as
  mean ± standard error.
